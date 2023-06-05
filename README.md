# HomeAssistant Component Helper

This is a C++ helper for defining HomeAssistant components using MQTT (PubSubClient).

It supports the following component classes:

- sensor
- binary_sensor
- switch
- connectivity

## Usage

First define a context using your pubsub client:

```c
PubSubClient client;
ComponentContext mqtt_context(client);
```

Define your HomeAssistant components.

```c
// This is a special component that mirrors the device availability state
HAAvailabilityComponent availability(mqtt_context);

// You can define any number of sensors:
HAComponent<Component::Sensor> sensor_temp(mqtt_context,  "Temp",  TEMP_SAMPLE_INTERVAL_MS, 0.f, SensorClass::Temperature);   // Temperature Sensor
HAComponent<Component::Sensor> sensor_humid(mqtt_context, "Humid", TEMP_SAMPLE_INTERVAL_MS, 0.f, SensorClass::Humidity);      // Humidity Sensor

// And any number of switches, with inline or global callbacks:
HAComponent<Component::Switch> switch_fan(mqtt_context, "fan", [](bool state) {
  Debug.println(state ? "Fan on!" : "Fan off!");
  digitalWrite(FAN_PIN, state);
});
```

To get things started, you need to connect to your MQTT client and use the HAAvailabilityComponent's last will topic, 
and set up a message received callback that forwards messages to the switch component:

```c

void onMessageReceived(char* topic, byte* payload, unsigned int length)
{
	payload[length] = '\0';
	Debug.println((char*)payload);

	String topic_s(topic);
	String payload_s((char*)payload);
	HAComponent<Component::Switch>::ProcessMqttTopic(topic_s, payload_s);
}

void setup() {
    ...
    
    // Metadata to report to HomeAssistant
    mqtt_context.device_name = "your_device";
    mqtt_context.friendly_name = "Your Device";
    mqtt_context.fw_version = "1.0.0";
    mqtt_context.manufacturer = "Manufacturer";
    mqtt_context.model = "Device Model";
    
    // Initialize each component with the above metadata
    HACompItem::InitializeAll();

    client.setCallback(onMessageReceived);
    
    while (!client.connected()) {
			String will_topic 		= HAAvailabilityComponent::inst->getWillTopic();
			const char* will_msg 	= HAAvailabilityComponent::OFFLINE;
			uint8_t will_qos 		= 0;
			bool will_retain 		= true;

			connected = client.connect(
				config.device_name, config.m_mqtt_user, config.m_mqtt_password, 
				will_topic.c_str(), will_qos, will_retain, will_msg);

    }
    
    // Now MQTT is connected, we can publish the components to HomeAssistant:
    sensor_temp.PublishConfig();
    sesnor_humid.PublishConfig();
    switch_fan.PublishConfig();
    availability.PublishConfig();
    availability.Connect();
    
}
```

Finally you can update your sensor readings as needed:
```c
void loop() {
    static unsigned long t_last = 0;
    
    client.loop();
    
    if ((millis() - t_last) > 1000) {
        float temperature = readTemperature();
        sensor_temp.Update(temperature);
        t_last = millis();
    }
}
```

