# <img width="40" alt="image" src="https://github.com/janaom/gcp-data-engineering-etl-with-composer-dataflow/assets/83917694/60f8f158-3bdc-4b3d-94ae-27a12441e2a3">  GCP Data Engineering Project: Automating Weather Forecast SMS Notifications with Composer/Airflow ⛅️

This small project was born out of curiosity and a real-life dilemma: I'm attending the KubeCon + CloudNativeCon Europe conference in just a few days, and I have no idea what the weather will be like in Paris 😅. So, I had this brilliant (or maybe slightly crazy) idea to create custom SMS updates for the Vilnius/Paris weather forecast. Imagine receiving a personal weather update while enjoying the conference!😎


Here is my solution. We will use GCP services:

- ⛅️ We'll connect to the weather API to fetch the next-day forecast for both Paris, France, and Vilnius, Lithuania.

- <img width="20" alt="image" src="https://github.com/janaom/gcp-de-project-connect-four-with-python-dataflow/assets/83917694/0887957d-db1b-4938-a9fa-f497fcebbeff"> The forecast data will be securely saved in a GCS bucket, ensuring easy access and safe storage.

- 💬 To keep you in the loop, we'll leverage Twilio to deliver the forecast as SMS messages to your phone every morning.

- <img width="18" alt="image" src="https://github.com/janaom/gcp-data-engineering-etl-with-composer-dataflow/assets/83917694/4c57cf42-15d3-4ba3-bad6-65b7fb9c5094"> The best part? We'll automate the entire process using Composer/Airflow, guaranteeing a smooth and effortless experience.

![20240310_213205](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/595e2c99-884d-485c-976b-144f7a0fa751)

# ⛅ OpenWeatherMap API

Visit https://openweathermap.org/ and create an account. You will receive an email titled 'OpenWeatherMap API Instruction' containing your API key, endpoint, an example API call, API documentation, and more. For additional information, please visit https://openweathermap.org/api.
Alternatively, you can find your API key on the website. Make sure to save this API key, as we'll need it later.

![image](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/3a62b649-1c36-435b-9d65-ea112fbfe64c)

To better understand the data available through the API, try to curl  the example API call provided. This will allow you to see all the available fields in the response.

```json
curl "http://api.openweathermap.org/data/2.5/weather?q=Paris,fr&appid=<Your_API_key>" | jq
{
  "coord": {
    "lon": 2.3488,
    "lat": 48.8534
  },
  "weather": [
    {
      "id": 500,
      "main": "Rain",
      "description": "light rain",
      "icon": "10d"
    }
  ],
  "base": "stations",
  "main": {
    "temp": 284.21,
    "feels_like": 283.54,
    "temp_min": 283.29,
    "temp_max": 284.92,
    "pressure": 992,
    "humidity": 83
  },
  "visibility": 10000,
  "wind": {
    "speed": 1.54,
    "deg": 0
  },
  "rain": {
    "1h": 0.18
  },
  "clouds": {
    "all": 100
  },
  "dt": 1709999501,
  "sys": {
    "type": 2,
    "id": 2041230,
    "country": "FR",
    "sunrise": 1709964959,
    "sunset": 1710006383
  },
  "timezone": 3600,
  "id": 2988507,
  "name": "Paris",
  "cod": 200
}
```

The response from the OpenWeatherMap API does not include temperature conversion to Celsius by default. To convert the temperature from Kelvin to Celsius, you can subtract 273.15 from the temperature value. Here's an updated version of the API call for Paris with the temperature converted to Celsius.

```json
curl "http://api.openweathermap.org/data/2.5/weather?q=Paris,fr&appid=<Your_API_key>" | jq '.main |= (.temp -= 273.15 | .feels_like -= 273.15 | .temp_min -= 273.15 | .temp_max -= 273.15)'
<...>
  "base": "stations",
  "main": {
    "temp": 11.04000000000002,
    "feels_like": 10.370000000000005,
    "temp_min": 10.140000000000043,
    "temp_max": 11.770000000000039,
    "pressure": 992,
    "humidity": 83
  },
<...>
```

Our DAG will only extract data about the minimum and maximum temperatures, as well as information about rain. Here's an example of the "Paris_FR_weather_20240309.json" file that will be delivered to the GCS bucket every morning.

```json
{"min_temp": 9, "max_temp": 10, "rain": true, "rain_description": "light rain"}
```

To retrieve the next day's weather forecast, we'll make use of the API endpoint that provides forecast data as part of its response. There's no need to worry about different units, as the temperatures will be returned in Celsius. This is because the API call includes the parameter units=metric which specifies that the temperature should be returned in Celsius.

```python
#Weather API URL for the forecast of the next day
    url = f"http://api.openweathermap.org/data/2.5/forecast?q={city},{country}&appid={api_key}&units=metric&cnt=2"

    #Make the API call
    response = requests.get(url)
    if response.status_code == 200:
        #Parse the response to get the min and max temperatures for the next day
        weather_data = response.json()
        next_day_data = weather_data['list'][1]  #Assuming the 2nd element is the next day
```

The cnt=2 parameter in the API call ensures that two forecasts are returned. By accessing weather_data['list'][1], we assume that the second element in the list corresponds to the forecast for the next day. For example, if the API is called on March 10th, the second element in the list will be the forecast for March 11th, as indicated by the "dt_txt": "2024–03–11 00:00:00".

```json
{
  "cod": "200",
  "message": 0,
  "cnt": 2,
  "list": [
    {
      "dt": 1710104400,
      "main": {
        "temp": 7.59,
        "feels_like": 6.85,
        "temp_min": 7.59,
        "temp_max": 9.36,
        "pressure": 999,
        "sea_level": 999,
        "grnd_level": 995,
        "humidity": 87,
        "temp_kf": -1.77
      },
      "weather": [
        {
          "id": 800,
          "main": "Clear",
          "description": "clear sky",
          "icon": "01n"
        }
      ],
      "clouds": {
        "all": 0
      },
      "wind": {
        "speed": 1.54,
        "deg": 245,
        "gust": 3.05
      },
      "visibility": 10000,
      "pop": 0.06,
      "sys": {
        "pod": "n"
      },
      "dt_txt": "2024-03-10 21:00:00"
    },
    {
      "dt": 1710115200,
      "main": {
        "temp": 7.57,
        "feels_like": 6.39,
        "temp_min": 7.54,
        "temp_max": 7.57,
        "pressure": 1000,
        "sea_level": 1000,
        "grnd_level": 996,
        "humidity": 83,
        "temp_kf": 0.03
      },
      "weather": [
        {
          "id": 802,
          "main": "Clouds",
          "description": "scattered clouds",
          "icon": "03n"
        }
      ],
      "clouds": {
        "all": 30
      },
      "wind": {
        "speed": 1.97,
        "deg": 239,
        "gust": 4.31
      },
      "visibility": 10000,
      "pop": 0.06,
      "sys": {
        "pod": "n"
      },
      "dt_txt": "2024-03-11 00:00:00"
    }
  ],
  "city": {
    "id": 2988507,
    "name": "Paris",
    "coord": {
      "lat": 48.8534,
      "lon": 2.3488
    },
    "country": "FR",
    "population": 2138551,
    "timezone": 3600,
    "sunrise": 1710051234,
    "sunset": 1710092875
  }
}
```

I highly recommend checking the data from the API, as I'm sure there are many interesting ways to use it; visit this page to see what's included in the Free plan: https://openweathermap.org/price#weather

# 💬 Twilio

Go to https://www.twilio.com and create an account. Open the Overview page, where you'll see messaging traffic from the past 30 days and recent message logs.

![image](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/76488048-e886-4162-a12c-a37de62b6793)

## WhatsApp/SMS notifications

Initially, I tried setting up the WhatsApp option. Click on 'Try WhatsApp'. You will see a similar page. Send a WhatsApp message and connect to WhatsApp Sandbox.

![Screenshot (1574)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/5f224b95-4b72-431f-ac9f-e80cef31f13d)


Click on Next step, send a template message to your WhatsApp number. 'To': is your real number. 'From': is Twilio number. If everything is fine you should receive the message from Twilio. Follow to the next step.

![Screenshot (1576)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/c60c890d-16a3-47cb-9258-bad3c7044756)


However, after 24 hours, my solution started to fail: 'Failed to send freeform message because you are outside the allowed window. If you are using WhatsApp, please use a Message Template.'

![Screenshot (1609)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/ea847f9f-c2b1-4b7f-ba88-ee031c0ec49c)


Twilio's WhatsApp messaging service allows freeform messages outside of message templates only within a 24-hour window. After that, you must use approved message templates. I tried different options but my template was rejected by WhatsApp. 

![Screenshot (1607)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/fe0d5388-6c0d-4bd2-98d4-8d90b3009fd3)

If you want to send notifications outside the 24-hour window without a WhatsApp Business API account, consider alternative messaging options like SMS or email. Twilio provides robust SMS and email messaging capabilities for notifications. I chose to use SMS. If you still want to try WhatsApp messages, setting up an account is easy. I added 2 versions of the code: `weather-forecast-whatsapp.py` and `weather-forecast-whatsapp-vars.py`.

```python
client.messages.create(body=message, from_=from_whatsapp_number, to=f"whatsapp:{to_whatsapp_number}")
```

Click on 'Send an SMS'. Follow the steps: 

Step 1: Recipients and Senders; 

Step 2: Sending messages. Enter your phone number and you will receive the test SMS. Trial accounts can only purchase 1 Twilio phone number.

![Screenshot (1595)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/32b4bb3c-0c6d-4eef-b17e-5927596ce2ea)

Check the console, where you will find your Account SID and Auth Token. It's essential to save these credentials, as they are required to authenticate your Twilio account and access its services in the future.

![Screenshot (1578)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/2f45a3fc-960c-4580-b3a4-5238e3edba76)

# ⭐ Composer 2

Create a Composer 2 environment. If this is your first time, remember to grant the Cloud Composer v2 API Service Agent Extension role to the Service Agent account.

![Screenshot (1559)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/e72eeee4-901a-476c-a9df-5c4d58016d97)

I'm using a Small environment, which typically takes 15 minutes to set up.

![Screenshot (1560)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/f4f1d458-dcea-4741-aa4d-de2680cb4bf1)

Ensure you install the twilio package on your Composer environment. If you don't, you won't be able to upload your DAG and will encounter a ```ModuleNotFoundError: No module named 'twilio'```.

![Screenshot (1582)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/a02d3346-d237-42cc-902e-ecad35452469)

You can either use variables in the code or save them in the Airflow UI. Both versions of the code are available: ```weather-forecast-sms.py``` and ```weather-forecast-sms-vars.py```.

![Screenshot (1598)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/278e1f52-c66b-4549-8299-73694cb35fd0)

Upload your DAG to the DAGs folder in the Composer environment.

![Screenshot (1599)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/c77427a2-6743-4ab7-a84c-e33077845367)

After a few minutes, you should see your DAGs in the Airflow UI.

![Screenshot (1605)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/d8456cc1-05b2-47ca-a19a-32f35e49d11b)

Trigger the DAG to test the solution.

![Screenshot (1603)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/6b35433b-863d-4532-897c-ae9d63138da2)

I'd like to highlight the trigger_rule I'm using for the second task. By setting trigger_rule=TriggerRule.ALL_DONE for the get_weather_data_vilnius task, you ensure that this task will execute after the get_weather_data_paris task has completed, regardless of its success or failure. This means that even if there's an exception, such as Failed to fetch weather data for Paris/FR you'll still receive weather data for Vilnius.

```python
get_weather_data_paris = PythonOperator(
    task_id="get_weather_data_paris",
    python_callable=get_weather_data_and_send_sms,
    op_args=["FR", "Paris"],
    dag=dag,
)

get_weather_data_vilnius = PythonOperator(
    task_id="get_weather_data_vilnius",
    python_callable=get_weather_data_and_send_sms,
    op_args=["LT", "Vilnius"],
    dag=dag,
    trigger_rule=TriggerRule.ALL_DONE,  #This will ensure the task runs regardless of upstream task success/failure
)
```

Verify that the weather forecast files are being saved in the GCS bucket as intended.

![Screenshot (1604)](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/aee3d79d-e5af-4205-ab68-c6c4f4601220)


Here's an example of the SMS/WhatsApp notifications I receive every morning at 9 AM with the weather forecast for Paris and Vilnius.

![MixCollage-11-Mar-2024-03-10-PM-7515](https://github.com/janaom/gcp-de-project-weather-forecast-sms-with-airflow/assets/83917694/153ded10-362b-459e-8143-6257ebaa8611)













