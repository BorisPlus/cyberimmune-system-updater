# Упражнение

__Описание__:

Инцидент типа [MitD](https://encyclopedia.kaspersky.com/glossary/man-in-the-disk-mitd).

Существует угроза безопасности, выраженная в подмене файла-обновления программы на _Storage_ в период между его проверкой со стороны _Verifier_ и его уставнокой _Updater_-ом.

## Идея №1

__Идеальная реализация__:

Введение цифровой подписи _Verifier_-а для файла-обновления и проверка этой цифровой подписи со стороный _Updater_-а при установке файла-обновления.

__Предположения безопасности__:

На вебинаре решили, что эта __тестовая__ задача поставлена для демонстрации участником курса базового понимания функционирования разработанного исходного процесса.

__Разумно-достаточная реализация__:

В предположении того, что _Manager_ и _Verifier_ доверенные, как собстенно и _Updater_, а каналы их взаимодействия также доверены и защищены от подмены передаваемых данных, то в схеме

![](./docs/report/diagrams/docs/report/sd/sd.png)

представляется достаточным через каналы _Verifier_-а и _Updater_-а с _Manager_-ом передать "подпись" _Verifier_-а для файла-обновления в _Updater_ для удостоверения последним в том, что файл, поступивший от _Storage_, действительно тот, что проверен _Verifier_-ом.

__Замечание ("если бы")__: В случае реализации технологии цифровой подписи, как она применяется в действительности (асимметричное шифрование и пр.), доверенностью _Manager_-а и каналов взаимодействия можно пренебречь. "Подпись" _Verifier_-а содержится в самом "запечатанном" файле-обновлении, без ее достоверности _Updater_ не выполнит установку.

### Реализация

Реализация основана на том, что циркулирующие в рассматриваемой системе данные помещаются в поля *details*. Правки соотвествующих значений направляют поток информации от модуля к модулю в соотвествии с "графом" политики взаимодействия.

Исходя из изложенного, в _Verifier_-е в _details_ ввел поле *'verified_digest'*. Его значение через _Manager_-а попадает к _Updater_-у. _Updater_ сличает полученную таким образом от _Verifier_-а цифровую подпись последнего и в случае достоверности проводит обновление.

__Замечания__:
* в исходном примере уже имеется функция `cleanup_extra_fields`*`, но я не стал деактивировать _del details['digest']_, так как логика работы мультисервисного приложения может опираться на отсуствие _details['digest']_;
* обновление вычисляет подпись полученных данных и сверяет с полученной подписью _Verifier_-а - _verifier_digest_ по принципу: "любая ошибка\исключение - подпись не валидна".

```python
def validate_verifier_digest(blob_data, verifier_digest) -> bool:
    try:
        payload_digest = sha256()
        update_payload = base64.b64decode(blob_data)
        payload_digest.update(update_payload)
        if verifier_digest == payload_digest.hexdigest():
            return True
    except:
        ...
    return False
```

Так, в частности, при _details['digest_alg'] != 'sha256'_, подпись не валидна, хотя данный аспект и так проверяется немного ранее по коду. Итог удостоверения в истинности цифровой подписи _Verifier_-а:

```python
if not validate_verifier_digest(details['blob'], details['verified_digest']):
        print('[error] verifier digest is not valid')
        return
```

Результат работы в случае компрометации будет выглядеть так:

```log
...
monitor_1           | [info] checking policies for event 77f93922-57f8-4aae-bd4d-854ef4df6891, storage->updater: blob_content
app_with_updater_1  | [info] handling event 77f93922-57f8-4aae-bd4d-854ef4df6891, storage->updater: blob_content
app_with_updater_1  | [info]===== EXECUTING UPDATE 77f93922-57f8-4aae-bd4d-854ef4df6891 ====
app_with_updater_1  | {'id': '77f93922-57f8-4aae-bd4d-854ef4df6891', 'operation': 'blob_content', 'target': 'app', 'deliver_to': 'updater', 'source': 'storage', 'authorized': True, 'verified': True, 'update_file_encoding': 'base64', 'blob_id': '77f93922-57f8-4aae-bd4d-854ef4df6891', 'verified_digest': '0c973a3ac6118770113ae356d5b1fd779af18f3b6a09077bd2602ea045b23f65', 'blob': 'UEsDBBQAAAAIAL2qklWblS+EGwEAANgBAAAGAAAAYXBwLnB5VZExb8MgEIV3fsUFL85ip1KnSpGaoVW7tFErtSMi9jmg4gMBThRV/e8FO7ESGODE4+Pdo1jUQ/D1TlONdAB3isoSY523PXRGhh/QvbM+wvNYwDyKy8GoYtI5WE+iUgiSPQqxnMV5U0DjUUYESZDVmkKU1CBjm+1WfD19fL6+vyUGv6tW1T1n7DGpKm+HiCWv+S1sIhcgI0SFgNSCs5oi1KzFDhQaY8vlw42BuUhOpDHQY+q1nbQsH3qMgyfo+Eu+Dt/Wm3YBG+eMbmTUluCAPuT198ryH78CH5Vu1BkUgI8+4JhBPPXdzgAa+h16pju4hAXr1LoQvdQkBJ+dF5Ce8wORpv35d3J6lTuNlseIBiqVDXHNVym5NHNW/1BLAQIUAxQAAAAIAL2qklWblS+EGwEAANgBAAAGAAAAAAAAAAAAAACkgQAAAABhcHAucHlQSwUGAAAAAAEAAQA0AAAAPwEAAAAA'}
app_with_updater_1  | [error] verifier digest is not valid
su-broker           | [2022-12-18 23:09:06,656] INFO [Controller id=1] Processing automatic preferred ...
```

### Пригодится

```bash
docker-compose build
make run
docker-compose logs -f
```

## Идея №2 - развитие №1

Из условий, описаных ранее, удалим из доверенных Manager-а. Соотвественно каналы связи с ним также становятся не довереннымы. Выступая посредниками и Storage, и Manager имеют возможность реализовать [MITM-атаку](https://encyclopedia.kaspersky.ru/glossary/man-in-the-middle-attack/).

Таким образом необходимо обеспечить достоверность того факта, что файл-обновления, устанавливаемый Updater-ом в действительности тот же самый, что проверялся Verifier-ом. Возвращаемся к технологии цифровой подписи. 

Предлагаю не реализовывать в учебном примере асимметричное шифрование, как это применяется в электронной цифровой подписи, а обойтись следующим предположением безопасности.

__Предположение безопасности__:

Сведения о секретном ключе шифрования доступны исключительно Updater-у и Verifier-у. Никакой иной процесс никогда не может получить значение ключа шифрования.

### Подпись файла-обновления

В результате проверки файла-обновления Verifier подписывает его по следующему алгоритму (хеширование с "солью", где "соль" - это и есть "секретный ключ" из предположения безопасности выше):

```python
SECRET_SALT = b'S0M3$3CR3TS#1T'

def set_up_data_digest(blob_data) -> str:
    if not blob_data:
        return None
    payload_digest = sha256()
    update_payload = base64.b64decode(blob_data)
    payload_digest.update(update_payload + SECRET_SALT)
    return payload_digest.hexdigest()

```

таким образом передавая по графу сообщений значение ЭЦП

```python
...
        details['verified_digest'] = set_up_data_digest(details['blob'])
```

Далее _Updater_, получая от _Storage_ файл-обновления, сверяет (он в любом случае должен реализовывать какую-то проверку) полученную предположительно от _Verifier_-а подпись `details['verified_digest']` со значением, полученным самостоятельно:

```python
    if not validate_verifier_digest(details['blob'], details['verified_digest']):
        print('[error] verifier digest is not valid')
        return
```

по такому же алгоритму (хеширование с "солью"):

```python
SECRET_SALT = b'S0M3$3CR3TS#1T'

def validate_verifier_digest(blob_data, verifier_digest) -> bool:
    try:
        payload_digest = sha256()
        update_payload = base64.b64decode(blob_data)
        payload_digest.update(update_payload + SECRET_SALT)
        if verifier_digest == payload_digest.hexdigest():
            return True
    except:
        ...
    return False
```

В случае компрометации файла-обновления со стороны _Storage_ или _Manager_-а им необходимо подделать в потоке данных и цифровую подпись _Verifier_-а `details['verified_digest']`. Но в силу нашего предположения безопасности, _Storage_ или _Manager_, даже имея сведения об используемом алгоритме шифрования (в частности по общему виду хеш-свертки), не владеют данными о подмешиваемой в алгоритме кодовой фразе - о "соли" `SECRET_SALT`.

__Заметка__: повысить устойчивость "соли" к компрометации возможно путем ее динамического генерирования с указанием времени жизни. Такой подход усложнит релизацию для демонстрации и разбора примера с недоверенным _Manager_-ом. Да и видится, что асимметричное шифрование для ЭЦП файла-обновления реализовать будет проще("дешевле" и "быстрее"), чем приведенное. Однако сохранность секретных ключей и для асимметричного шифрования также будет необходимо безоговорочно обеспечивать, и внести в предположения безопасности.

Результат работы в случае компрометации будет выглядеть так:

```log
manager_1           | update event: {'id': '02498814-2c39-4573-b7b9-2da07fe16647', 'operation': 'download_file', 'target': 'app', 'url': 'http://file_server:5001/download-update/app-update.zip', 'deliver_to': 'downloader', 'digest': 'e774a61bf55c0b34a196a93583a27bb84814a0473c99d1183c0c9bff5cffaa44', 'digest_alg': 'sha256'}
manager_1           | 172.24.0.1 - - [19/Dec/2022 21:30:52] "POST /update HTTP/1.1" 200 -
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, manager->downloader: download_file
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, manager->downloader: download_file
downloader_1        | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, manager->downloader: download_file
file_server_1       | 172.24.0.8 - - [19/Dec/2022 21:30:52] "GET /download-update/app-update.zip HTTP/1.1" 200 -
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, downloader->manager: download_done
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, downloader->manager: download_done
manager_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, downloader->manager: download_done
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, manager->storage: commit_blob
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, manager->storage: commit_blob
storage_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, manager->storage: commit_blob
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, storage->manager: blob_committed
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, storage->manager: blob_committed
manager_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, storage->manager: blob_committed
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, manager->verifier: verification_requested
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, manager->verifier: verification_requested
verifier_1          | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, manager->verifier: verification_requested
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, verifier->storage: get_blob
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, verifier->storage: get_blob
storage_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, verifier->storage: get_blob
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, storage->verifier: blob_content
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, storage->verifier: blob_content
verifier_1          | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, storage->verifier: blob_content
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, verifier->manager: handle_verification_result
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, verifier->manager: handle_verification_result
manager_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, verifier->manager: handle_verification_result
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, manager->updater: proceed_with_update
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, manager->updater: proceed_with_update
app_with_updater_1  | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, manager->updater: proceed_with_update
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, updater->storage: get_blob
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, updater->storage: get_blob
storage_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, updater->storage: get_blob
monitor_1           | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, storage->updater: blob_content
monitor_1           | [info] checking policies for event 02498814-2c39-4573-b7b9-2da07fe16647, storage->updater: blob_content
app_with_updater_1  | [info] handling event 02498814-2c39-4573-b7b9-2da07fe16647, storage->updater: blob_content
app_with_updater_1  | [info]===== EXECUTING UPDATE 02498814-2c39-4573-b7b9-2da07fe16647 ====
app_with_updater_1  | [error] verifier digest is not valid 
```


Результат работы в случае успеха:

```log
monitor_1           | [info] checking policies for event 7ed076ff-ca22-46a8-9a1f-875fc7fdfc0f, storage->updater: blob_content
app_with_updater_1  | [info] handling event 7ed076ff-ca22-46a8-9a1f-875fc7fdfc0f, storage->updater: blob_content
app_with_updater_1  | [info]===== EXECUTING UPDATE 7ed076ff-ca22-46a8-9a1f-875fc7fdfc0f ====
app_with_updater_1  | Archive:  tmp/7ed076ff-ca22-46a8-9a1f-875fc7fdfc0f
app_with_updater_1  |   inflating: ../app/app.py           
app_with_updater_1  | [info] update result code 0
app_with_updater_1  |  * Detected change in '/app/app.py', reloading
app_with_updater_1  |  * Restarting with stat
app_with_updater_1  |  * Debugger is active!
app_with_updater_1  |  * Debugger PIN: 116-223-660
```