
# Инструкция по настройке автоматического воспроизведения видео через DLNA в Home Assistant

Этот проект позволяет настроить автоматическое воспроизведение видео с DLNA-сервера на устройствах, поддерживающих медиа-контент через Home Assistant. Мы используем REST API для взаимодействия с DLNA-сервером, чтобы получать информацию о последних видеороликах и автоматически воспроизводить их на смарт-устройствах.

## Требования

1. **Home Assistant** - требуется настроенная система Home Assistant.
2. **DLNA-сервер** - устройство, которое предоставляет доступ к видео через DLNA (например, NAS, медиаплеер или роутер с поддержкой DLNA).
3. **Подключенный медиаплеер** (например, смарт-ТВ), поддерживающий воспроизведение видео через Home Assistant.

## Шаги по настройке

### 1. Получение данных DLNA-сервера

Для начала необходимо получить URL вашего DLNA-сервера, который будет использован для обращения к его API.

- **Смените IP-адрес** сервера в следующей строке:

```yaml
resource: http://192.168.2.1:8200/ctl/ContentDir
```

Замените `192.168.2.1` на IP-адрес вашего DLNA-сервера.

- **Убедитесь, что порт** вашего сервера настроен правильно (по умолчанию это `8200`, но может отличаться).

### 2. Конфигурация сенсора в Home Assistant

Вам нужно будет добавить сенсор, который будет делать запросы к вашему DLNA-серверу и получать ссылку на последнее видео.

1. Откройте файл `configuration.yaml` в вашей папке конфигурации Home Assistant.
2. Вставьте следующий код в этот файл:

```yaml
sensor:
  - platform: rest
    resource: http://192.168.2.1:8200/ctl/ContentDir
    method: POST
    payload: >
      <?xml version="1.0" encoding="utf-8"?>
      <s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
        <s:Body>
          <u:Browse xmlns:u="urn:schemas-upnp-org:service:ContentDirectory:1">
            <ObjectID>2$FF0</ObjectID>
            <BrowseFlag>BrowseDirectChildren</BrowseFlag>
            <Filter>*</Filter>
            <StartingIndex>0</StartingIndex>
            <RequestedCount>10</RequestedCount>
            <SortCriteria></SortCriteria>
          </u:Browse>
        </s:Body>
      </s:Envelope>
    headers:
      Content-Type: text/xml; charset=utf-8
      SOAPAction: urn:schemas-upnp-org:service:ContentDirectory:1#Browse
    name: last_dlna_file_url
    scan_interval: 60
    value_template: >
      {% set result = value | regex_findall('<res.*?>(http://.*?)</res>') %}
      {% if result %}
        {{ result[0] }}
      {% else %}
        No video found
      {% endif %}
```

**Что нужно изменить:**

- **IP-адрес**: Замените `http://192.168.2.1:8200` на адрес вашего DLNA-сервера.
- **Секцию `ObjectID`**: Если у вашего сервера другой `ObjectID`, замените его на соответствующий.

### 3. Настройка автоматизации для воспроизведения видео

Для того, чтобы видео автоматически воспроизводилось на вашем медиаплеере (например, на смарт-ТВ), добавьте следующую автоматизацию в файл `automations.yaml`:

```yaml
automation:
  - alias: "Автовоспроизведение последнего видеофайла"
    trigger:
      platform: state
      entity_id: sensor.last_dlna_file_url
      to: "{{ states('sensor.last_dlna_file_url') }}"
    condition:
      condition: template
      value_template: "{{ states('sensor.last_dlna_file_url') | trim != 'No video found' }}"
    action:
      - service: media_player.play_media
        target:
          entity_id: media_player.smart_tv
        data_template:
          media_content_id: "{{ states('sensor.last_dlna_file_url') }}"
          media_content_type: video/mp4
```

**Что нужно изменить:**

- **`media_player.smart_tv`**: Замените `media_player.smart_tv` на ID вашего медиаплеера (например, `media_player.living_room_tv`).

### 4. Перезагрузка Home Assistant

После того как вы добавите изменения в конфигурационные файлы:

1. Перезапустите Home Assistant, чтобы применить изменения:
   - Перейдите в `Конфигурация > Управление сервером > Перезагрузить`.
   
2. После перезагрузки, сенсор `sensor.last_dlna_file_url` будет отслеживать последнее видео с вашего DLNA-сервера и автоматически воспроизводить его на вашем медиаплеере.

### 5. Проблемы и отладка

- Если сенсор отображает **"No video found"**, проверьте IP-адрес и настройки вашего DLNA-сервера. Убедитесь, что сервер доступен и корректно настроен.
- Если видео не воспроизводится, проверьте, что медиа-плеер в Home Assistant правильно настроен и поддерживает воспроизведение видео.

## Примечания

- Этот код был протестирован для DLNA-серверов, использующих протокол SOAP. Если ваш сервер использует другие методы, вам может потребоваться адаптировать код.
- Пример воспроизведения видео настроен для типа `video/mp4`. Если ваши видео имеют другой формат, убедитесь, что в `media_content_type` указан правильный тип MIME для вашего формата (например, `video/x-matroska` для `.mkv`).

