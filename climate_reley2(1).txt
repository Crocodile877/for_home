import machine
import network
import dht
import time
import ujson
import urequests

dht_sensor = dht.DHT11(machine.Pin(2))
# GPIO number to which the LED is connected
LED_PIN = 2 
# Initializing GPIO for LED
led = machine.Pin(LED_PIN, machine.Pin.OUT)

rele_fan = machine.Pin(22,machine.Pin.OUT) #GPIO number to which the relay is connected
rele_mois = machine.Pin(23,machine.Pin.OUT) #GPIO number to which the relay is connected
rele_fan.value(1)
rele_mois.value(1)


# network connection
ssid = 'name of your network'
password = 'password'
BOT_TOKEN = 'your bot token'
# Chat ID with which the bot will interact
CHAT_ID = 0000000  #Replace with your chat ID


# Wi-Fi
wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(ssid, password)
while not wlan.isconnected():
    pass


#Function for sending messages in Telegram
def send_telegram_message(text):
    url = f'https://api.telegram.org/bot{BOT_TOKEN}/sendMessage'
    data = {'chat_id': CHAT_ID, 'text': text}
    response = urequests.post(url, json=data)
    response.close()

send_telegram_message(wlan.ifconfig()[0])
send_telegram_message('ready')

# global variable to store the last message ID
last_update_id = 0
# Main loop
while True:
    try:
        # Request to receive updates from Telegram
        url = f'https://api.telegram.org/bot{BOT_TOKEN}/getUpdates?offset={last_update_id + 1}'
        response = urequests.get(url)
        data = ujson.loads(response.text)
        response.close()

        # Processing updates
        for update in data['result']:
            if 'message' in update and 'text' in update['message']:
                message = update['message']['text'].lower()
                chat_id = update['message']['chat']['id']

                # checking message novelty by ID
                if update['update_id'] > last_update_id:
                    last_update_id = update['update_id']  #Update latest message ID

                    if message == '/fan_on':
                        rele_fan.value(0)
                        send_telegram_message('fan-on')
                    elif message == '/fan_off':
                        rele_fan.value(1)
                        send_telegram_message('fan-off')
                    elif message == '/moistening_on':
                        rele_mois.value(0)
                        send_telegram_message('mois-on')
                    elif message == '/moistening_off':
                        rele_mois.value(1)
                        send_telegram_message('mois-off')
                    elif message == '/temperature':
                        dht_sensor.measure()
                        temp_celsius = dht_sensor.temperature()
                        send_telegram_message(temp_celsius)
                    elif message == '/humidity':
                        dht_sensor.measure()
                        humidity_percentage = dht_sensor.humidity()
                        send_telegram_message(humidity_percentage)

        # Sleep mode before next checking for updates
        time.sleep(5)

    except Exception as e:
        send_telegram_message('error' : {}'.format(str(e)))


