# Home Assistant DLNA Auto Play

Этот проект включает автоматизацию для Home Assistant, которая позволяет автоматически воспроизводить последнее видео, найденное через DLNA на медиаплеере, таком как Smart TV.

## Описание

В Home Assistant настроен сенсор `rest`, который запрашивает список видеофайлов через DLNA. Когда новый файл появляется, автоматически запускается воспроизведение на медиаплеере.

## Требования

- Home Assistant
- Подключение к DLNA-серверу
- Медиаплеер (например, Smart TV)

## Конфигурация

### Сенсор:

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
