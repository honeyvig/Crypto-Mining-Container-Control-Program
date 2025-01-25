# Crypto-Mining-Container-Control-Program
I'm building a 1MW Bitcoin and it needs to operate remotely.

The mine needs a "brain" to read data, take some simple actions and send notifications.

Goal - a continuous loop program monitoring and acting on a signal from our power provider that will turn the miners/fans on and off as needed to save money.

Background - our power provider gives us a low rate if we turn off our miners when power demand is high. They have a website with a number that tells us when to turn off our miners. Since this will probably happen during the hottest and coldest times, it is important to monitor the temperatures and control the fan system as well.

This program will need to run on a dedicated RaspberryPi4 8G that will have a fixed IP with other fixed IP devices on the same network - except the 1 external webpage that will need to be scraped.

Consider this a looping state machine that reads data to records state and then take actions based on the states.

Logging and notification is very important - this is a remote mine - 
All activities and state changes are logged - extensive logging
Notifications & Heartbeat are sent to telegram bot
Warnings are sent via bot & email
Emergencies are sent via bot, email & Text

A simple dashboard is needed to show all current states (and trends in the future)

A debugging dashboard is needed to be able to SET states for testing.

A config file is required to read configuration variables from
LoopInterval = 2 minutes

The Loop
- read config.txt for interval in minutes
- read saved states from time series db
- read PowerSignal and set state
	- read Cornbelt Webpage (web scrape, no API) for Signal
- read Fan&Temp and set state
	- read RoomAlert via API for temps & alerts
	- read IotaWatt/eMonCMS via API for fan wattage
- write states to time series db
- take actions
	- read states
		- reduce miner power
			- API call to foreman.mn
			- API call to PDU if needed
		- resume miner power
			- API call for foreman.mn
			- API call to PDU if needed
		- turn fans off
			- API call to Fan Relays
		- turn fans on
			- API call to Fan Relays
	- log actions
- write states
- update web dashboard output file
- report heartbeat
- exit until called next by cron?


4 phases

1 - Main loop running on raspi4/8G
- cron loop
- config read, write & edit app
- state machine read/write and ready for additions - time series db for later easy reporting/Grafana?
- telegram, email & text alerts
- simple web dashboard  to see states, action & logs
- github distribution onto the raspi

2 - INPUTS read & saved at interval
- RoomAlert Temp & Humidity
- CBW Fans web API for fan control
- Cornbelt Power Signal from cbpower.coop
- IotaWatt for fan wattage
- SmartPDU for miner wattage
- edit state for testing
- All logging and alerts engaged and tested

3 - ACTIONS
- Miners Off
- Miners On
- Fans Off  1 - 9
- Fans On 1 - 9

PS - I have this project partially completed (but unfinished/tested) and you are welcome to use any or all portions of that code.To create the program you’ve described, we'll break it down into the necessary components and phases, keeping the state machine architecture intact. This program will run as a daemon on your Raspberry Pi 4, executing periodic checks and actions based on external signals. The logic will be implemented in a Python script that integrates with your various devices via API calls, scrapes data from the web, sends notifications via Telegram, email, and SMS, and maintains logs.

I'll also break it down by phases to give you a clearer understanding of what each part does.

### Phase 1: **Main Loop**
The main loop is responsible for continuously executing every set interval (e.g., 2 minutes), reading various states, taking actions, and sending notifications if necessary.

### Step-by-Step Python Code for Main Loop

1. **Install Dependencies:**
You'll need to install some dependencies on your Raspberry Pi. These are for interacting with APIs, web scraping, sending notifications, and logging.

```bash
pip install requests smtplib telegram schedule pandas
```

2. **Create Configuration File:**
The configuration file (`config.txt`) will hold essential variables such as loop interval, power signal URL, temperature API, etc.

**config.txt:**
```
LoopInterval=2
PowerSignalURL=http://cbpower.coop/power-signal
FanAPIURL=http://roomalert.local/api/fan
TempAPIURL=http://roomalert.local/api/temp
IotaWattAPI=http://iotawatt.local/api/fan-wattage
SmartPDUAPI=http://smartpdu.local/api/miner
TelegramBotToken=your-telegram-bot-token
TelegramChatID=your-chat-id
EmailSMTPServer=smtp.gmail.com
EmailSMTPPort=587
EmailUser=your-email@gmail.com
EmailPassword=your-email-password
```

### 3. **Python Script to Handle Logic:**

Here’s a simplified version of the Python code implementing the above logic. We'll start with reading configuration, scraping data, and sending notifications.

```python
import time
import requests
import smtplib
import telegram
import pandas as pd
from datetime import datetime
import json

# Read configuration file
def read_config():
    config = {}
    with open("config.txt", "r") as file:
        for line in file:
            if line.strip() and '=' in line:
                key, value = line.split('=')
                config[key.strip()] = value.strip()
    return config

# Log function
def log_event(message):
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    with open("mine_log.txt", "a") as log_file:
        log_file.write(f"{timestamp} - {message}\n")

# Send notifications
def send_telegram_message(message, config):
    bot = telegram.Bot(token=config['TelegramBotToken'])
    bot.send_message(chat_id=config['TelegramChatID'], text=message)

def send_email(message, config):
    with smtplib.SMTP(config['EmailSMTPServer'], config['EmailSMTPPort']) as server:
        server.starttls()
        server.login(config['EmailUser'], config['EmailPassword'])
        server.sendmail(config['EmailUser'], config['EmailUser'], message)

# Web Scrape Power Signal from the Cornbelt webpage
def get_power_signal(config):
    response = requests.get(config['PowerSignalURL'])
    if response.status_code == 200:
        # Parse the signal (assuming it's a JSON or simple text response)
        data = response.text
        return data.strip()
    else:
        log_event("Error fetching power signal")
        return None

# Fetch RoomAlert Temperature and Humidity data
def get_temp_and_humidity(config):
    response = requests.get(config['TempAPIURL'])
    if response.status_code == 200:
        data = response.json()
        temp = data['temperature']
        humidity = data['humidity']
        return temp, humidity
    else:
        log_event("Error fetching temperature data")
        return None, None

# Fetch Fan wattage from IotaWatt
def get_fan_wattage(config):
    response = requests.get(config['IotaWattAPI'])
    if response.status_code == 200:
        data = response.json()
        wattage = data['fan_wattage']
        return wattage
    else:
        log_event("Error fetching fan wattage")
        return None

# Fetch miner wattage from SmartPDU
def get_miner_wattage(config):
    response = requests.get(config['SmartPDUAPI'])
    if response.status_code == 200:
        data = response.json()
        wattage = data['miner_wattage']
        return wattage
    else:
        log_event("Error fetching miner wattage")
        return None

# Take action based on power signal and temperature
def take_action(power_signal, temp, wattage, config):
    if power_signal == 'High' and temp > 80:
        # If power is high and temperature is too high, turn off miners and fans
        log_event("Power is high and temperature is high. Turning off miners and fans.")
        send_telegram_message("Power is high and temperature is high. Turning off miners and fans.", config)
        # Turn off miners and fans (API calls)
        # Example: requests.post(f"{config['FanAPIURL']}/turn_off")
        # Example: requests.post(f"{config['MinerAPIURL']}/turn_off")
        
    elif power_signal == 'Low' and temp < 60:
        # If power is low and temperature is low, turn on miners and fans
        log_event("Power is low and temperature is low. Turning on miners and fans.")
        send_telegram_message("Power is low and temperature is low. Turning on miners and fans.", config)
        # Turn on miners and fans (API calls)
        # Example: requests.post(f"{config['FanAPIURL']}/turn_on")
        # Example: requests.post(f"{config['MinerAPIURL']}/turn_on")

# Main Loop
def main():
    config = read_config()
    interval = int(config['LoopInterval']) * 60  # Convert minutes to seconds

    while True:
        log_event("Starting loop...")

        # Fetch data
        power_signal = get_power_signal(config)
        temp, humidity = get_temp_and_humidity(config)
        fan_wattage = get_fan_wattage(config)
        miner_wattage = get_miner_wattage(config)

        # Save states to time series db (For example, pandas DataFrame)
        data = pd.DataFrame({
            'timestamp': [datetime.now()],
            'power_signal': [power_signal],
            'temperature': [temp],
            'humidity': [humidity],
            'fan_wattage': [fan_wattage],
            'miner_wattage': [miner_wattage]
        })
        data.to_csv("states.csv", mode='a', header=False, index=False)

        # Take necessary actions based on data
        take_action(power_signal, temp, fan_wattage, config)

        # Wait before next loop
        time.sleep(interval)

if __name__ == "__main__":
    main()
```

### Explanation:

1. **Configuration Reading (`read_config`):**
   - Reads variables such as power signal URL, fan control URL, API endpoints, and notification credentials from a configuration file.
   
2. **Logging (`log_event`):**
   - Logs events and state changes into a text file with timestamps. This will keep track of all important actions.

3. **Notification System (`send_telegram_message`, `send_email`):**
   - Sends Telegram messages and emails for different actions (e.g., turning miners off, high power signal, etc.).

4. **Web Scraping (`get_power_signal`):**
   - Scrapes the power signal from the Cornbelt website (you may need to adjust this based on the actual HTML structure of the webpage).

5. **Data Fetching (Temperature, Wattage):**
   - Fetches room temperature, fan wattage, and miner wattage from APIs.

6. **State Machine & Actions (`take_action`):**
   - Based on the fetched signals (e.g., high power, low power), the script will turn miners or fans on/off.

7. **Time Series DB (`states.csv`):**
   - Saves the states into a CSV file, which can later be analyzed or used with Grafana for visualization.

### Phase 2: **Testing and Debugging**
- **Test State Transitions**: You can create additional functionality in the program that allows you to test specific states.
- **Setup on Raspberry Pi**: Install the script as a cron job to run at regular intervals. Ensure the Raspberry Pi is set up with a fixed IP and network configurations.

### Phase 3: **Final Actions**
- Ensure the correct integration of your APIs for miner and fan controls.
- Implement a proper web dashboard (perhaps using **Flask** or **Dash** for Python) to visualize current states and trends.

### Notes:
- You'll need to adapt specific API calls for your devices (e.g., miner and fan control).
- Logging and notifications are essential for monitoring a remote mine.
- A full-fledged web dashboard can be implemented using **Flask** or **Django**.
