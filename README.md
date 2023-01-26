## Unit testing in python through time using FreezeGun

Any software you write should be properly unit tested to ensure they work as intended - it helps with debugging and detecting any unexpected consequences of updating code later on and making sure your code can deal with edge cases. 

But how can you test functions whose behaviour is dependent on the current time?

Some examples of this:
- You have a logger that logs the current time
- A user logs into an application and you show them a notification wishing them a happy birthday on their birthday
- You want to send staff notifications, only if they're currently scheduled to be on shift
- You have an application that shows different recommendations based on time of day and day of week

Enter `freezegun` - A python package that can help you unit test by freezing time at a time given in your unit tests.

This package can be installed using pip:

```sh
pip install freezegun
```

## An example

The National Grid publishes an API that lets you know the Carbon Intensity in $gCO_2/kWh$ during any 30 minute period of the day.

You can view any 30 minute period by visiting:

https://api.carbonintensity.org.uk/intensity/date/{date}/{period}

We want to write a function that, at any given time, at any given locale, will give us the current date and period in the UK so we can fill the parameters in the URL above with the current `date` and `period`.

So if you visit:

https://api.carbonintensity.org.uk/intensity/date/2023-01-26/1

You should see:

```json
{ 
  "data":[{ 
    "from": "2023-01-26T00:00Z",
    "to": "2023-01-26T00:30Z",
    "intensity": {
      "forecast": 158,
      "actual": 152,
      "index": "moderate"
    }
  }]
}
```

However, if you visit a date in the summer, for example:

https://api.carbonintensity.org.uk/intensity/date/2022-06-01/1

You'll see:

```json
{ 
  "data":[{ 
    "from": "2022-05-31T23:00Z",
    "to": "2022-05-31T23:30Z",
    "intensity": {
      "forecast": 256,
      "actual": 261,
      "index": "high"
    }
  }]
}
```

Note that the first time is at 00:00 in UTC but the second time is 23:00 UTC. This is because the period "1" denotes the first half an hour period in the day in UK local time.

In the winter this is GMT (UTC+00) and in summer this is BST (British Summer Time; UTC+01).

So in winter we have:

| Date       | Period | Local Time       | UTC Time         |
|------------|--------|------------------|------------------|
| 2023-01-26 | 1      | 2023-01-26 00:00 | 2023-01-26 00:00 |
|            | 2      | 2023-01-26 00:30 | 2023-01-26 00:30 |
|            | ...    |                  |                  |
|            | 48     | 2023-01-26 23:30 | 2023-01-26 23:30 |

And in summer we have:

| Date       | Period | Local Time       | UTC Time         |
|------------|--------|------------------|------------------|
| 2022-06-01 | 1      | 2022-06-01 00:00 | 2022-05-31 23:00 |
|            | 2      | 2022-06-01 00:30 | 2022-05-31 23:30 |
|            | ...    |                  |                  |
|            | 48     | 2022-06-01 23:30 | 2023-06-01 22:30 |



### Edge Cases

Of course as this API is in local time, there are edge cases we have to consider caused by daylight savings time.

When the clocks "go forward" in spring, we are missing periods and there are only 46 periods in the day:

| Date       | Period | Local Time       | Local Timezone | UTC Time         |
|------------|--------|------------------|----------------|------------------|
| 2022-03-27 | 1      | 2022-03-27 00:00 | GMT            | 2022-03-27 00:00 |
|            | 2      | 2022-03-27 00:30 | GMT            | 2022-03-27 00:30 |
|            | 3      | 2022-03-27 02:00 | BST            | 2022-03-27 01:00 |
|            | 4      | 2022-03-27 02:30 | BST            | 2022-03-27 01:30 |
|            | ...    |                  |                |                  |
|            | 46     | 2022-03-27 23:30 | BST            | 2022-03-27 22:30 |

And when the clocks "go back" in autumn, we have additional periods and there are 50 periods in the day:

| Date       | Period | Local Time       | Local Timezone | UTC Time         |
|------------|--------|------------------|----------------|------------------|
| 2022-10-30 | 1      | 2022-10-30 00:00 | BST            | 2022-10-29 23:00 |
|            | 2      | 2022-10-30 00:30 | BST            | 2022-10-29 23:30 |
|            | 3      | 2022-10-30 01:00 | BST            | 2022-10-30 00:00 |
|            | 4      | 2022-10-30 01:30 | BST            | 2022-10-30 00:30 |
|            | 5      | 2022-10-30 01:00 | GMT            | 2022-10-30 01:00 |
|            | 6      | 2022-10-30 01:30 | GMT            | 2022-10-30 01:30 |
|            | ...    |                  |                |                  |
|            | 50     | 2022-10-30 23:30 | GMT            | 2022-10-30 23:30 |


## Write Test Cases

So in a true test-driven development approach, let's start with our test cases.

We're going to use the `freeze_time` context manager:

```python
with freeze_time(utc_dt, tz_offset=0)
```

To freeze time at any given datetime in UTC.

We're going to try out our winter times, summer times, Daylight Savings Time (DST) start and end edge cases and some other random times to make sure we cover a good range:


```python
import datetime
import unittest

from freezegun import freeze_time


class TestGetCurrentDateAndPeriod(unittest.TestCase):
    def run_test(self, test_case):
        utc_dt, expected_date, expected_period = test_case
        # Parse date string
        expected_date = datetime.datetime.strptime(
            expected_date,
            '%Y-%m-%d'
        ).date()
        # Freeze time and run test
        with freeze_time(utc_dt, tz_offset=0):
            local_date, period_in_day = get_current_date_and_period()
            self.assertEqual(local_date, expected_date)
            self.assertEqual(period_in_day, expected_period)

    def test_winter_periods(self):
        # "Current" UTC Datetime, Expected Date, Expected Period
        winter_test_cases = (
            ('2023-01-26 00:00', '2023-01-26', 1),
            ('2023-01-26 00:30', '2023-01-26', 2),
            ('2023-01-26 23:30', '2023-01-26', 48)
        )
        for test_case in winter_test_cases:
            self.run_test(test_case)
    
    def test_summer_periods(self):
        # "Current" UTC Datetime, Expected Date, Expected Period
        summer_test_cases = (
            ('2022-05-31 23:00', '2022-06-01', 1),
            ('2022-05-31 23:30', '2022-06-01', 2),
            ('2022-06-01 00:00', '2022-06-01', 3),
            ('2022-06-01 22:30', '2022-06-01', 48)
        )
        for test_case in summer_test_cases:
            self.run_test(test_case)
    
    def test_dst_start_periods(self):
        # "Current" UTC Datetime, Expected Date, Expected Period
        dst_start_test_cases = (
            ('2022-03-27 00:00', '2022-03-27', 1),
            ('2022-03-27 00:30', '2022-03-27', 2),
            ('2022-03-27 01:00', '2022-03-27', 3),
            ('2022-03-27 01:30', '2022-03-27', 4),
            ('2022-03-27 22:30', '2022-03-27', 46),
            ('2022-03-27 23:00', '2022-03-28', 1)
        )
        for test_case in dst_start_test_cases:
            self.run_test(test_case)
    
    def test_dst_end_periods(self):
        # "Current" UTC Datetime, Expected Date, Expected Period
        dst_end_test_cases = (
            ('2022-10-29 23:00', '2022-10-30', 1),
            ('2022-10-29 23:30', '2022-10-30', 2),
            ('2022-10-30 00:00', '2022-10-30', 3),
            ('2022-10-30 00:30', '2022-10-30', 4),
            ('2022-10-30 01:00', '2022-10-30', 5),
            ('2022-10-30 01:30', '2022-10-30', 6),
            ('2022-10-30 23:30', '2022-10-30', 50),
            ('2022-10-31 00:00', '2022-10-31', 1)
        )
        for test_case in dst_end_test_cases:
            self.run_test(test_case)
    
    def test_get_current_date_and_period(self):
        # "Current" UTC Datetime, Expected Date, Expected Period
        other_test_cases = test_cases = (
            ("2021-01-01 00:01", "2021-01-01", 1),
            ("2021-01-01 12:17", "2021-01-01", 25),
            ("2021-06-11 12:17", "2021-06-11", 27),
            ("2021-06-11 12:29", "2021-06-11", 27),
            ("2021-06-11 12:30", "2021-06-11", 28),
            ("2021-06-11 12:31", "2021-06-11", 28),
            ("2022-02-14 23:59", "2022-02-14", 48),
            ("2022-04-05 23:59", "2022-04-06", 2),
        )
        for test_case in other_test_cases:
            self.run_test(test_case)
```

## Write Functions

Now that we've got our test cases set up to test the `get_current_date_and_period` function we'll need to create this functionality.

We have some other helper functions in order to achieve this but I've provided docstrings in reStructuredText format to describe what these functions do.


```python
import math
from typing import Tuple, Optional

import pytz


def get_utc_time_start_of_date(local_date: datetime.date,
                               local_tz: str = 'Europe/London'
                              ) -> datetime.datetime:
    """
    Given a Date and a Local Timezone, this function will return
    the Datetime in UTC at the start of this date in the given
    timezone.
    
    :param local_date: A date to get the UTC start time for
    :param local_tz: The Olson local timezone as a string, default
        is Europe/London
    :type local_date: datetime.date
    :type local_tz: str
    :returns: Timezone-aware UTC datetime
    :rtype: datetime.datetime
    
    :Example:
    >>> local_date = datetime.date(2021, 6, 1)
    >>> get_utc_time_start_of_date(local_date)
    datetime.datetime(2021, 5, 31, 23, 0, tzinfo=<UTC>)
    """
    time_start = datetime.time(0, 0)
    dt_start = datetime.datetime.combine(local_date, time_start)
    local_tz = pytz.timezone(local_tz)
    local_dt_start = local_tz.localize(dt_start)
    utc_dt_start = pytz.utc.normalize(local_dt_start)
    return utc_dt_start


def get_local_date(utc_dt: datetime.datetime,
                   local_tz: str = 'Europe/London'
                  ) -> datetime.date:
    """
    Given a UTC datetime and a Local Timezone, this function will
    return the date in the Local Timezone.
    
    :param utc_dt: Datetime in UTC (Timezone optional)
    :param local_tz: The Olson local timezone as a string, default
        is Europe/London
    :type utc_dt: datetime.datetime
    :type local_tz: str
    :returns: Local Date for the given UTC Timezone
    :rtype: datetime.date
    
    :Example:
    >>> example_dt = datetime.datetime(2022, 5, 1, 12, 30)
    >>> get_local_date(example_dt)
    datetime.date(2022, 5, 1)
    
    >>> utc_dt_diff_local_date = datetime.datetime(2022, 6, 1, 23, 45)
    >>> get_local_date(utc_dt_diff_local_date)
    datetime.date(2022, 6, 2)
    """
    if utc_dt.tzinfo is None:
        utc_dt = pytz.utc.localize(utc_dt)
    local_tz = pytz.timezone(local_tz)
    local_dt = local_tz.normalize(utc_dt)
    return local_dt.date()


def get_period_in_the_day(utc_dt: datetime.datetime,
                          local_date: datetime.date,
                          local_tz: str = 'Europe/London') -> int:
    """
    Given a Datetime in UTC, and the local date, this function
    will return the 1-indexed half an hour period of the day
    
    :param utc_dt: Datetime in UTC
    :param local_date: The local date of this datetime
    :param local_tz: The Olson local timezone of the local date as a string;
        default is Europe/London
    :type utc_dt: datetime.datetime
    :type local_date: datetime.date
    :type local_tz: str
    
    :Example:
    >>> utc_dt = datetime.datetime(2023, 1, 1, 0, 0)
    >>> local_date = datetime.date(2023, 3, 1)
    >>> get_period_in_the_day(utc_dt, local_date)
    1
    
    >>> utc_dt = datetime.datetime(2022, 7, 12, 14, 23)
    >>> local_date = datetime.date(2022, 7, 12)
    >>> get_period_in_the_day(utc_dt, local_date)
    31
    
    >>> utc_dt = datetime.datetime(2022, 6, 1, 23, 45)
    >>> local_date = datetime.date(2022, 6, 2)
    >>> get_period_in_the_day(utc_dt, local_date)
    2
    """
    if utc_dt.tzinfo is None:
        utc_dt = pytz.utc.localize(utc_dt)
    utc_dt_start = get_utc_time_start_of_date(local_date, local_tz)
    time_diff = (utc_dt - utc_dt_start).total_seconds()
    period_in_day = math.floor(time_diff / (60 * 30)) + 1
    return period_in_day


def get_current_date_and_period(local_tz: Optional[str] = 'Europe/London'
                               ) -> Tuple[datetime.date, int]:
    """
    :param local_tz: The Olson local timezone of the local date as a string;
        default is Europe/London
    :type local_tz: str
    
    :Example:
    >>> get_current_date_and_period()
    (datetime.date(2023, 1, 26), 27)
    """
    utc_time_now = pytz.utc.localize(
        datetime.datetime.utcnow()
    )
    local_date = get_local_date(utc_time_now, local_tz)
    period_in_day = get_period_in_the_day(
        utc_time_now,
        local_date,
        local_tz
    )
    return local_date, period_in_day
```

## Run Tests

Now that we've got our tests set up, we'll run our Unit Test test cases:

```python
from io import StringIO


def run_unit_tests(unit_test_class):
    stream = StringIO()
    runner = unittest.TextTestRunner(stream=stream, verbosity=2)
    runner.run(unittest.makeSuite(unit_test_class))
    stream.seek(0)
    print(stream.read())


run_unit_tests(TestGetCurrentDateAndPeriod)
```

```
test_dst_end_periods (__main__.TestGetCurrentDateAndPeriod) ... ok
test_dst_start_periods (__main__.TestGetCurrentDateAndPeriod) ... ok
test_get_current_date_and_period (__main__.TestGetCurrentDateAndPeriod) ... ok
test_summer_periods (__main__.TestGetCurrentDateAndPeriod) ... ok
test_winter_periods (__main__.TestGetCurrentDateAndPeriod) ... ok

----------------------------------------------------------------------
Ran 5 tests in 0.517s

OK
```

And we can see that all of our tests passed!

FreezeGun has a range of other functionality including the ability to restart time using the `tick` argument, automatically increment time every time the time is read using `auto_tick_seconds` argument and the ability to manually increment time.

I fully recommend exploring it the next time you have any functionality that uses the current date and time.
