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

