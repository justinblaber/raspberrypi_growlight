```python
# default_exp raspberrypi_growlight
```

This is a simple bare-bones webserver run on my raspberry pi to turn my dwarf citrus tree's grow light on at sunrise and off at sunset.

# Imports


```python
# export
import datetime
import sys
import time

import apscheduler.schedulers.background
import astral
import astral.sun
import pytz
import RPi.GPIO as GPIO
from flask import Flask
```

# Config

### Raspberry Pi


```python
# export
PIN_GPIO = 17
GPIO.setwarnings(False)
GPIO.setmode(GPIO.BCM)         # Broadcom Chip
GPIO.setup(PIN_GPIO, GPIO.OUT) # Output 
```

Test out gpio stuff manually


```python
GPIO.output(PIN_GPIO, True)
```


```python
GPIO.output(PIN_GPIO, False)
```

### Timezone

Needed to localize `datetime.now()`


```python
# export
TIMEZONE = pytz.timezone('US/Eastern')
```

### Server

Set interval in seconds to check schedule


```python
# export
INTERVAL = 1
```

# `Growlight`

Growlight `on()` will turn on grow light and `off()` will turn off grow light


```python
# export
class Growlight:
    def on(self):
        GPIO.output(PIN_GPIO, True)
    
    def off(self):
        GPIO.output(PIN_GPIO, False)
```

Test to see if growlight turns on and off


```python
growlight = Growlight()
```


```python
growlight.on()
```


```python
time.sleep(1)
```


```python
growlight.off()
```

# `Schedule`

Given an input time, `Schedule` will return `'ON'` if grow light should be on and `'OFF'` if growlight should be off


```python
# export
class Schedule:
    def get_status(T):
        return 'OFF' 
```

Sunlight schedule will turn on during sunlight


```python
# export
class SunlightSchedule(Schedule):
    def __init__(self, city):
        self.city = city
        
    def get_status(self, T):
        s = astral.sun.sun(self.city.observer) # Get current status of sun
        if s['sunrise'] < T < s['sunset']: return 'ON'
        else:                              return 'OFF'
    
    def __repr__(self):
        return f'sunlight schedule for {self.city.name}.'
```

Make a schedule based on lowell 


```python
# export
class LowellSunlightSchedule(SunlightSchedule):
    def __init__(self):
        super().__init__(city=astral.LocationInfo(name='Lowell',
                                                  region='USA',
                                                  timezone='Eastern',
                                                  latitude=42.640999,  
                                                  longitude=-71.316711))
```

Test out schedule


```python
schedule = LowellSunlightSchedule()
schedule
```




    sunlight schedule for Lowell.




```python
T = TIMEZONE.localize(datetime.datetime.now())
schedule.get_status(T)
```




    'ON'



Add some time to see if scheduler works


```python
schedule.get_status(T + datetime.timedelta(hours=10))
```




    'OFF'



# `GrowlightScheduler`

Given a `Schedule` and a `GrowLight`, `GrowlightScheduler` will turn on or off the growlight based on the schedule at a given `interval`.


```python
# export
class GrowlightScheduler:
    def __init__(self, schedule, growlight, interval=INTERVAL):
        self.schedule  = schedule
        self.growlight = growlight
        self.interval  = interval
        self.scheduler = apscheduler.schedulers.background.BackgroundScheduler()
        self.scheduler.add_job(self.job, 'interval', seconds=self.interval)
        
    def job(self):
        T = TIMEZONE.localize(datetime.datetime.now())
        status = self.schedule.get_status(T)
        if   status == 'ON':  self.growlight.on()
        elif status == 'OFF': self.growlight.off()
        else:                 raise RuntimeError(f'Unknown status: {status}')
    
    def start(self):    self.scheduler.start()
    def shutdown(self): self.scheduler.shutdown()
```


```python
schedule = LowellSunlightSchedule()
growlight = Growlight()
scheduler_growlight = GrowlightScheduler(schedule, growlight)
```


```python
scheduler_growlight.start()
```


```python
scheduler_growlight.shutdown()
```


```python
growlight.off()
```

# Flask app

Create flask app with the following API:

* `ON`     - shutsdown scheduler and turns growlight on
* `OFF`    - shutsdown scheduler and turns growlight off
* `START`  - starts scheduler
* `STOP`   - shutsdown scheduler
* `STATUS` - returns current status


```python
# export
app = Flask(__name__)
schedule = LowellSunlightSchedule()
growlight = Growlight()
scheduler_growlight = GrowlightScheduler(schedule, growlight)
STATUS = None
```


```python
# export
def _status():
    return f'Growlight status: {STATUS}'
```


```python
# export
@app.route('/ON/', methods=['GET'])
def ON():
    global STATUS
    scheduler_growlight.shutdown()
    growlight.on()
    STATUS = 'ON'
    return _status()
```


```python
# export
@app.route('/OFF/', methods=['GET'])
def OFF():
    global STATUS
    scheduler_growlight.shutdown()
    growlight.off()
    STATUS = 'OFF'
    return _status()
```


```python
# export
@app.route('/START/', methods=['GET'])
def START():
    global STATUS
    scheduler_growlight.start()
    STATUS = str(scheduler_growlight.schedule)
    return _status()
```


```python
# export
@app.route('/STOP/', methods=['GET'])
def PAUSE(): 
    global STATUS
    scheduler_growlight.shutdown()
    STATUS = 'stopped ' + str(scheduler_growlight.schedule)
    return _status()
```


```python
# export
@app.route('/STATUS/', methods=['GET'])
def STATUS():
    return _status()
```

For testing purposes change name


```python
__name__ = '__notebook__'
```

By default, start grownlight scheduler when server starts up


```python
# export
if __name__ == '__main__':
    START()
    app.run(host='0.0.0.0', port=8080)
```

# Build


```python
!nbdev_build_lib --fname raspberrypi_growlight.ipynb
```

    Converted raspberrypi_growlight.ipynb.



```python
!jupyter nbconvert --to markdown --output README raspberrypi_growlight.ipynb
```

    [NbConvertApp] Converting notebook raspberrypi_growlight.ipynb to markdown
    [NbConvertApp] Writing 6391 bytes to README.md

