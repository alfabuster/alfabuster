Вопрос 7 - Инфраструктура

Восстановите сертификат и закрытый ключ (выпущен MOYKA ROOT CA), сохраненный в локальном хранилище машины Windows (доступном в оснастке certlm.msc). Для решения задания вам дан дамп файлов https://github.com/testdsec/SOH2026/tree/main/task_7, необходимых для восстановления. В качестве доказательства решения укажите сертификат и закрытый ключ в формате PEM, а также пошаговую инструкцию по их восстановлению.


Первое, что нужно сделать это достать пароль DPAPI_SYSTEM, он должен находиться в ветке SECURITY реестра Windows. DPAPI_SYSTEM  лежит не просто так, а зашифрованный. А ключом к нему является SysKey, спрятанный в SYSTEM.

```
lsadump::secrets /system:C:\Users\Den\Downloads\machine-cert-dump\Windows\System32\config\SYSTEM /security:C:\Users\Den\Downloads\machine-cert-dump\Windows\System32\config\SECURITY
```

Mimikatz выдаёт секрет в нескольких форматах, нам нужен full.

<img width="1920" height="960" alt="Пароль DPAPI_SYSTEM в mimikatz" src="https://github.com/user-attachments/assets/15c16e1b-d631-467b-a620-5c1473b9c16f" />


Теперь нужно расшифровать мастер-ключ.

```
dpapi::masterkey /in:"C:\Users\Den\Downloads\machine-cert-dump\Windows\System32\Microsoft\Protect\S-1-5-18\c6859e4a-bfd8-46bf-86e7-2574c4b59eb2" /system:a0e9a91ca5fd7a82473c11e002ab613bad8c906e39bf9ba8af68b32aba14b83eb5b093b551f5bb56
```

<img width="1920" height="913" alt="Расшифровка Мастер-ключа" src="https://github.com/user-attachments/assets/37c181af-aa89-463c-b764-8b2bf2265828" />


Расшифровка закрытого ключа (RSA)

Файл fc58e716dd94cece30c087d4db914620_e1d345bd-ebc1-4d50-8b78-3f3e787e0953 это зашифрованный контейнер. Будем использовать модуль capi для расшифровки RSA ключа.

```
dpapi::capi /in:"C:\Users\Den\Downloads\machine-cert-dump\ProgramData\Microsoft\Crypto\RSA\MachineKeys\fc58e716dd94cece30c087d4db914620_e1d345bd-ebc1-4d50-8b78-3f3e787e0953" /masterkey:43d0f1e158b63d8e5e992cb886a142bdce9f7c0404417f39699e7f53e73183f7bbf090a918bde8cb1c9821dac4094f5a2c2ad18e0642d4c0667a7f5c9b2d81a7
```
<img width="1920" height="912" alt="Расшифровка закрытого ключа (RSA)" src="https://github.com/user-attachments/assets/ac1d4ccd-67a8-4a31-9d96-0a95db4f9828" />


Mimikatz расшифрует Binary Large OBject (бинарный большой объект) и покажет структуру RSA ключа.

Извлечение сертификата из SOFTWARE

Закрытый ключ без сертификата бесполезен. Публичную часть (сертификат) нужно достать из реестра. Поэтому нужно загрузить ветку в SOFTWARE в редактор реестра.

<img width="1920" height="961" alt="Загружаем ветку SOFTWARE в редактор реестра" src="https://github.com/user-attachments/assets/cc982789-77a6-4599-850f-109ea958e40b" />

<img width="1920" height="963" alt="Blob в редакторе реестра" src="https://github.com/user-attachments/assets/2dd2dfb8-cb64-4cf8-a798-af53c4ca1d4e" />

Просто так экспортнуть блоб из редактора реестра не получится, поэтому нужно использовать либо скрипт на питоне, либо воспользоваться Powershell.

```powershell
$registryPath = "HKLM:\Den\Microsoft\SystemCertificates\MY\Certificates\DB74D352754B27D58F8DA91B772F223718922619"

PS C:\Users\Den> $blob = (Get-ItemProperty -Path $registryPath).Blob

[System.IO.File]::WriteAllBytes("C:\Users\Den\Downloads\mimikatz\x64\cert.bin", $blob)
```
<img width="1920" height="963" alt="Экспорт Blob в файл через PowerShell" src="https://github.com/user-attachments/assets/8a760096-433a-4ec5-ae49-2d8c28f801a9" />


Теперь у нас есть бинарник. Он содержит "мусор" от Microsoft (заголовки свойств). Нам нужно извлечь из него настоящий сертификат.

Все сертификаты X.509 в мире описаны стандартом ASN.1 (Abstract Syntax Notation One) — это способ записывать сложные структуры данных в бинарном виде. В кодировке DER 30 - это SEQUENCE (последовательность), а 82 - 2 байта длины. Поэтому любой X.509 сертификат в мире начинается с байтов 30 82. Все что находится до этих байтов нужно удалить.

Откроем этот файл в программе HxD.

<img width="1920" height="962" alt="HxD удаляем мусор из бинарника" src="https://github.com/user-attachments/assets/a1c5913a-2694-4868-9f86-a172f8e31164" />


Найдем байты 30 82 и удаляем, все что до них, сохраняем в формате der. Получим валидный сертификат.

<img width="1920" height="963" alt="Получим валидный сертификат" src="https://github.com/user-attachments/assets/23cc0492-8c86-4f6e-a7af-fde29180e922" />


Осталось сконвертировать сертификат в формат pem.

```
.\openssl.exe x509 -in C:\Users\Den\Downloads\mimikatz\x64\cert.der -out C:\Users\Den\Downloads\mimikatz\x64\cert.pem
```

И закрытый ключ.

```
.\openssl.exe" rsa -inform PVK -in C:\Users\Den\Downloads\mimikatz\x64\dpapi_exchange_capi_0_{9D947440-8500-4E0D-81D9-EFBA81B82D97}.keyx.rsa.pvk -out C:\Users\Den\Downloads\mimikatz\x64\key.pem
```

```cert.pem
-----BEGIN CERTIFICATE-----
MIIFUTCCAzkCFCl/h76wv/zF2+BDk5d2yBs0G1K5MA0GCSqGSIb3DQEBCwUAMGcx
CzAJBgNVBAYTAlJVMRcwFQYDVQQIDA5TdC4gUGV0ZXJzYnVyZzEXMBUGA1UEBwwO
U3QuIFBldGVyc2J1cmcxDjAMBgNVBAoMBU1PWUtBMRYwFAYDVQQDDA1NT1lLQSBS
T09UIENBMB4XDTI2MDQwMTAyMDIwMFoXDTI3MDQwMTAyMDIwMFowYzELMAkGA1UE
BhMCUlUxFzAVBgNVBAgMDlN0LiBQZXRlcnNidXJnMRcwFQYDVQQHDA5TdC4gUGV0
ZXJzYnVyZzEOMAwGA1UECgwFTU9ZS0ExEjAQBgNVBAMMCUlWQU5PVi1OQjCCAiIw
DQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBANYGhIjnROymnT9O534afYhk0gq5
8/RAR2f/X7WLE4BdARnCrCy1SNWwqbLfjkNyLM8cLUEjYKl11kjPeHlfO6CLNMVG
3M8M6P4emz6frYPXJnEDcJQ3BsuMPH0zhIBK9N9gqvwxtyRWNSkcw9GqTQqofLQ2
e3sxGxat2TM1+giI818mgaZ97mrxAyo1r9OFlZKOwRBMEPgQcPXuoBZYE0WxSwwP
KzN3cz3pgzfCGzUdFCIhNY6z9+F0X773g2ccIMRASCFQ9YtJC9mYu4MATPUyr6+s
EEdZ0qD53s7jmwfVTZYAiwdu7vMnb8J+RZ2nU7MVM4rXsf6wlt/UuKipbq4SW6ZD
UaoX8Owjge0a/bT4KK3eqGobQRq+TkD78SjncdVOksibcKR439UkbKT6pPGTDcC3
EslK7XHayAIUf+IzT/I18QFM5bOayT5TsUUAEUzV3sg8HrH+J0PZybi/FWskqnRe
QWJHxjwmpE0mxTnoOEQgrEj0Txg0NuOml2dqNgpp3S550efKXB6YA4UG2+jEVU7A
bGLFVliqFdOWlWgphiYsF3W6In4B6mlcnCuFz7DAtAOObhTQ8D1knnPig3p09izr
NLqbO/vtZZ1bU7o00CtfQC1k1ibzN7ipTJlaa479fnkAPoHr+I1E1OW/rprIwcPJ
2ILyGvkZAviz7L9/AgMBAAEwDQYJKoZIhvcNAQELBQADggIBADvL15FUWQfYnVqG
YocnafnOL/A9l8vorwgHVeOFMTh1nAfCQniKQrrlsOLM8vcvcE0QuBJVavoPt4Fi
/smvFQlAs23Y8EuRvSDqBuU9mDgJdejGmfVbFgcCll5PtCysja7sXSSSGFtfe9Iw
3JT5iMpuOV8NFP7JkhQHG1iVyxh9l3BSeeKKNlDLmMwmrPtExzoWsBwwK6hjrGPp
jRTCpTyTYL9+JynCj+rKI0jcu2TSH6rRx41nPGZwHB5j+OxU0uFlFcWHtOXoH998
PW3CimQDLey8CR+ujJr1l+QJNTWrAu6fg4fEeveMirR+8M5661ujGnPT/X7zIIVN
a42JW3zktanHrib1+xqVmmpUzdux665+H02YL7HxP5SyKZr/JD3u0oTRJN0j2Yqh
oDWWLZYLKkytGJdyxBV0Kh83+kHoFQ4Zkgej2XtWZ7shKpgIAQ6tUffI0OGXdJ0P
FmlHHCRW1ZEF8IA71lUVkVpIjW76OwD5Edts0GK1e8FuO1LLNNojV5liJnWyhxLd
86/0m0AF/8muyKxuQ/swQby1DvoSmlA/XodfnB72+Se90rUVxk/mEEXQ0DimvQ3p
qQY/cudMklmhqExGclWOjY8CuljPaNVxY5UXe8ztwgJ2vT/tjGTGIXUhLnSW9XkI
DtrDNyYck03iwRvwNPa9SQ1TgGfj
-----END CERTIFICATE-----

```

```key.pem
-----BEGIN PRIVATE KEY-----
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
cnfbssQPP7oCQDoIXDzYjN2L5PM4zmiI
-----END PRIVATE KEY-----

```
