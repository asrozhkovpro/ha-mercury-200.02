# Home Assistant (ZigBee) <-> Меркурий200.02
!! Сам компонент, как и оформление репозитория находится в разработке !!  

Custom Home Assistant integration for obtaining the data from Energy meter __Mercury 200.02__. This Integration is dedicated for Russia and CIS users, as I believe, no one outside uses this type of meters. However, if you need documentation in English, please, let me know in issues.  

Бытовые счетчики электроэнергии __Меркурий 200.02__ обладают цифровым интерфейсом на базе __RS-485__, который позволяет удаленно считывать показания расхода электроэнергии. 
Данная интеграция позволяет считывать показания и отображать их в Home Assistant через [zigbee2mqtt](https://www.zigbee2mqtt.io). Для подключения электросчетчика к Zigbee сети необходим модем. О нем - ниже.   
<img src="https://raw.githubusercontent.com/MenshikovDmitry/ha-mercury-200-integration/master/images/mercury200.02.png">  
  
Вдохновлен постом [Smart Home 53](https://zen.yandex.ru/media/id/5f5bea45267c75477b342dab/integraciia-schetchika-merkurii-200-v-home-assistant-chast-1-5f959fae4dcc5c613c00a83a). Расшифровка протокола Mercury200 [в этом репо](https://github.com/mrkrasser/MercuryStats). Спасибо Авторам!
  
**ОТКАЗ ОТ ОТВЕТСТВЕННОСТИ. Дорогие друзья, автор этого компонента как и Вы - энтузиаст и для меня это - хобби. Компонент распространяется под лицензией MIT. Если У Вас есть вопросы или пожелания, пишите в Issues. Я постараюсь найти время и помочь, но обещать ничего не могу. У меня все работает :-D.**

## Содержание
 _В разработке_

## Установка

### Через [HACS](https://hacs.xyz/) (рекомендуется)
_Раздел в разработке_  
Добавьте ссылку на этот репозиторий в HACS и Установите компонент.

### Вручную (не рекомендуется)

_В разработке_

## Конфигурация

После установки компонента возможно понадобится перезагрузка. Далее необходимо добавить в файл конфигурации _configuration.yaml_: 

```yaml
# Electricity Counter Mercury200.02
mercury200:
  - type: mercury200.02
    device_serial: "XXXXXXXX" 
    topic: "zigbee2mqtt/electricity_counter"
```
__Примечания:__  
 - Пока поддерживается только mercury200.02.
 - серийный номер счетчика можно посмотреть на самом счетчике, а так же в ЛК Мосэнергосбыт или другого оператора
 - _'electricity_counter'_ - это _friendly_name_ передающего устройства (модема) в zigbee2mqtt.

## Использование
### Сущности
После перезагрузки в Entities появится список сущностей, относящихся к Счетчику:  
  
<img src="https://raw.githubusercontent.com/MenshikovDmitry/ha-mercury-200-integration/master/images/entities.png">  
Изначально значения сущностей будут недоступны либо будут нулевыми. Это нормально.  

### Сервисы
Чтобы получить значения со счетчика, необходимо отправить на него запрос. Для этого в компоненте предусмотрен сервис _submit_command_, формирующий запрос в байтах:  

```yaml
    - service: mercury200.submit_command
      data:
        device_id: "XXXXXXXX" # Серийный номер счетчика
        command: get_energy
```
 - get_energy: отправляет запрос на получение значений тарифов T1-T4
 - get_status: отправляет запрос на получение текущих показаний сети: Сила тока, напряжение, потребляемая мощность

Получив запрос, электросчетчик отвечаечает байт-строкой, которая транслируется zigbee модемом в zigbee2mqtt. Компонент проводит расшифровку сообщения и обновляет значения сущностей.  
Для регулярного обновления значений можно добавить в _automations.yaml_:

```yaml
# Collect data from electricity counter
- alias: Mercury T1 - T3
  trigger:
    - platform: time_pattern
      minutes: "/57"
  action:
    - service: mercury200.submit_command
      data:
        device_id: "04025230"
        command: get_energy

- alias: Mercury status
  trigger:
    - platform: time_pattern
      minutes: "/5"
  action:
    - service: mercury200.submit_command
      data:
        device_id: "04025230"
        command: get_status
```
### Карточка
Ниже мой вариант карточки в yaml для LoveLace. Для этого необходим компонент [multiple entity row](https://github.com/benct/lovelace-multiple-entity-row). В качестве дополнительной информации, а так же для автоматической отправки показаний счетчика в личный кабинет, рекомендую использовать компонент [ЛК "Интер РАО"](https://zzun.app/repo/alryaz-hass-lkcomu-interrao). Автору больше Спасибо!      
<img src="https://raw.githubusercontent.com/MenshikovDmitry/ha-mercury-200-integration/master/images/lovelace.png">
<details>
<summary> Пример карточки в yaml для Lovelace </summary>

```yaml
type: entities
entities:
  - entity: sensor.mercury200_XXXXXXXX_power
    name: Status
    secondary_info: last-updated
    icon: mdi:lightning-bolt
    type: custom:multiple-entity-row
    entities:
      - icon: mdi:refresh
        tap_action:
          action: call-service
          service: mercury200.submit_command
          service_data:
            device_id: 'XXXXXXXX'
            command: get_status
        name: false
      - entity: sensor.mercury200_XXXXXXXX_voltage
        name: false
  - entity: sensor.mercury200_XXXXXXXX_t3
    name: Counter (kW*h)
    icon: mdi:gauge
    type: custom:multiple-entity-row
    show_state: false
    secondary_info: last-updated
    entities:
      - icon: mdi:refresh
        tap_action:
          action: call-service
          service: mercury200.submit_command
          service_data:
            device_id: 'XXXXXXXX'
            command: get_energy
      - entity: sensor.mercury200_XXXXXXXX_t1
        name: T1
        unit: false
      - entity: sensor.mercury200_XXXXXXXX_t2
        name: T2
        unit: false
      - entity: sensor.mercury200_XXXXXXXX_t3
        name: T3
        unit: false
  - entity: sensor.mes_XXXXX_XXX_XX_meter_XXXXXXXX
    name: '-> Mosenergo'
    type: custom:multiple-entity-row
    secondary_info: last-changed
    attribute: last_indications_date
    format: date
    icon: mdi:cloud-upload
    entities:
      - entity: sensor.mes_XXXXX_XXX_XX_account
        name: Debth

```
</details>

### Модем
**Раздел в Разработке**  
**!!Данная версия модема пока не протестирована!!**  
Модем необходим для подключения Счетчика электроэнергии к zigbee2mqtt.  
В папке modem лежит файл Gerber, необходимый для печати плат, а также файл прошивки. Пайка минимальна.
Наполнение:  

| **Элемент** | **Номинал** | **Ссылка** |
|-------------|-------------|------------|
| корпус      | 0603        |        |
| R1          | 0603        |        |
| R2          |             |        |
| R3          |             |        |
| C1, C2      |             |        |
| U1          |             |        |


### Помогите разработке
- [ ] Проверка кофигурации с _homeassistant.helpers.config_validation_  
- [ ] Зашить в компонент периодическое обновление данных (Сейчас это делается вручную или через _automations.yaml_)
- [ ] По непонятной мне причине показания счетчиков в карточке обновляются в течение 30 секунд после фактического получения данных (К примеру, после ручного обновления через иконку в карточке).
- [ ] Кастомная прошивка для модема (сейчас она на базе конфигуратора [PTVO](https://ptvo.info/). За что ему большое Спасибо!)
- [ ] Поддержка других счетчиков
- [ ] Доработка документации
  - [ ] Прошивка
  - [ ] Установка в ручном режиме
  - [ ] Подключение модема к счетчику электроэнергии
- [ ] Донат. Вы всегда можете поддержать авторов zigbee2mqtt, PTVO, Home assistant и других контрибьюторов. Потом может и я заслужу) 
