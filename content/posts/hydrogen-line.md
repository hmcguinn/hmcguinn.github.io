---
title: "Backyard Hydrogen Line Astronomy"
date: 2022-01-02T23:19:26-05:00
draft: true
---

We can utilize a cheap (< $250) software-defined radio (SDR) backyard setup to observe the 21cm hydrogen line and probe the structure of our galaxy! One of the coolest parts of radio astronomy is that you can do it during the day! It can be a little more convenient to make observation while the sun is up, and for our case specifically, the Milky Way is overhead for a decent chunk of the day.

# Background 

![spin flip](/img/spin-flip-scaled.png)

Neutral hydrogen atoms occasionally undergo a change in energy state from higher to lower, which gives rise to the [21cm Hydrogen line](https://en.wikipedia.org/wiki/Hydrogen_line). A Hydrogen atom has two possible spin configurations; the first is a slightly higher energy state where the proton and electron have the same spin, with the magnetic field of the electron reversed. In the second, the particles have the opposite spin with the same magnetic field. Given enough time, a Hydrogen atom in the first state will spontaneously flip its spin orientation to the second state. The energy difference between the two is given off by a photon with a wavelength of 21cm-- our Hydrogen line. 

\\[ \lambda = \frac{1}{v} \dot \\, c = \frac{h}{E} \approx \frac{4.1357 \dot 10^{-15} eV \\, s}{5.87433 \dot 10^{-6} eV} \\, \dot \\, 2.9979 \dot 10^{8} \\, m \\, s^{-1} \approx 0.21106 \\, m = 21.106 \\, cm \\]

While the spin-flip transition is rare (taking ~10 million years to occur for a single atom), there is a lot of hydrogen in space. Hydrogen is concentrated in the spiral arms of the Milky Way and especially in the galactic center (near Sagittarius). The Hydrogen line is very narrow, centered on 1420.405 Mhz. Using this knowledge, we can image different arms of the Milky Way and observe the delta between the observed and expected frequency. This can be used to calculate the radial speed of the different arms of the galaxy! 

# Setup 

I followed [this guide from RTL-SDR.com](https://www.rtl-sdr.com/cheap-and-easy-hydrogen-line-radio-astronomy-with-a-rtl-sdr-wifi-parabolic-grid-dish-lna-and-sdrsharp/) to setup my equipment. I used a [1 meter parabolic dish](https://www.amazon.com/Premiertek-Directional-High-Gain-Parabolic-ANT-GRID-24DBI/dp/B005M8KU3W/ref=pd_sbs_1/137-2260590-1166642?pd_rd_w=lxWGy&pf_rd_p=0a3ad226-8a77-4898-9a99-63ffeb1aef90&pf_rd_r=JFQETBH5P545YV1JS3TV&pd_rd_r=0238062a-5707-4b87-a79a-ac321cfcc06d&pd_rd_wg=g7M7Y&pd_rd_i=B005M8KU3W&psc=1), [Sawbird H1 low noise amplifier (LNA)](https://www.amazon.com/dp/B07XPV9RX2?ref=nb_sb_ss_w_as-ypp-rep_ypp_rep_k0_1_7&amp&crid=1KSDU4I4AMG3P&amp&sprefix=sawbird), [50Ω terminator](https://www.amazon.com/gp/product/B00BXUYDMM), and a [RTL-SDR](https://www.rtl-sdr.com/). The total cost for the equipment came to just over $200, not including the RTL-SDR and raspberry pi I already owned. 

I connected my RTL-SDR to the LNA directly, and connected the LNA to the parabolic dish. The SDR was then connected to my raspberry pi using a USB->USB cable. 

# Software

I used [Virgo](https://github.com/0xCoto/Virgo) to process my raw data. It bundles together some tools to plan observations, use calibration data, filter out RFI, and creates some pretty plots. 

Below is a short, two-minute exposure I took. You'll notice the calibrated spectrum looks very different from the average spectrum. The three large peaks in the average spectrum are an artifact of the low-noise amplifier's filter shape. In the calibrated spectrum, the small peak just to the right of the hydrogen line is the emission given off by the Milky Way! The sharp line to the right is some RFI. 

![hydrogen plot](/img/hydrogen-line.png)

# Doppler Shifts

We can calculate the radial velocity of our observed target using the doppler method!

\\[ \frac{\Delta \lambda}{\lambda} = \frac{v}{c} \\]

I don't have quite the amount of data I'd like to calculate doppler shifts for all arms of the Milky Way, but rest assured this method does work! PhysicsOpenLab has a [great article on this](https://physicsopenlab.org/2020/09/08/measurement-of-the-milky-way-rotation/). 

While I made my first batch of observations manually, it'd be ideal to automate them so we can have our observatory run continuously! However, there isn't always a suitable target overhead to observe. We need some way to determine whether or not we should make an observation!

# Automating the Observatory

We'd like to automatically make observations when some portion of the Milky Way is overhead. Luckily, Virgo provides some handy tools to let us do this.

First, we'll need to get our right ascension and declination to figure out where in the sky our radio telescope is pointing. To make things simple, we'll assume our telescope is pointing straight upwards.  

```
altitude = 90
azimuth = 0
ra, dec = virgo.equatorial(altitude,azimuth,latitude,longitude)
```

Once we know where in the sky we're looking, we'll want to translate this to where we're looking relative to the galaxy. 

```
_ , latitude = virgo.galactic(coords[0],coords[1])
```

Because our telescope is pointing straight up, we're only concerned with the latitude. Our telescope has a beamwidth of approximately 8 degrees so we'll want to observe if:

```
latitude > -8/2 and latitude < 8/2
```

This only takes into account the center of the galactic coordinates, the Milky Way has an apparent width of around ~30 degrees. We can adjust our observation condition to:

```
latitude >= -30/2 and latitude <= 30/2
```

Putting it all together, this is the scipt I used to automate my observatory.

```
import virgo
from datetime import datetime
from subprocess import Popen
import sys

try:
    latitude = longitude = 0 # your lat/long
    coords = virgo.equatorial(90,0,latitude,longitude)
    coords = virgo.galactic(coords[0],coords[1])
    lat = coords[1]
    if lat >= -30 and lat <= 30:
        prefix = './Observations/' + datetime.today().strftime('%Y-%m-%d-%H-%M')  + '/'
        print(prefix)
        cmd = 'mkdir -p ' + prefix
        p = Popen(cmd, shell=True)

        obs = {
            'dev_args': 'rtl,bias=1',
            'rf_gain': 30,
            'if_gain': 25,
            'bb_gain': 18,
            'frequency': 1420e6,
            'bandwidth': 2.4e6,
            'channels': 2048,
            't_sample': 1,
            'duration': 60
        }

        # Calibrate the data
        print("Calibrating the dish, duration: " + str(obs['duration']) + ' seconds' )
        timestamp = datetime.today().strftime('%Y-%m-%d-%H-%M-%S')
        cal_file_name = str(prefix + 'cal-' + timestamp + '.dat')
        print("Saving in file: " + cal_file_name)
        print('')

        virgo.observe(obs_parameters=obs, obs_file=cal_file_name)
        print('')
        input("Move the satellite dish to the Galactic Plane")
        print('')
        # Observe
        obs['duration'] = 600*3 # time in seconds
        print("Starting observation, duration: " + str(obs['duration']) + ' seconds')
        timestamp = datetime.today().strftime('%Y-%m-%d-%H-%M-%S')
        obs_file_name = str(prefix+ 'observe-' + timestamp + '.dat')
        print("Saving in file: " + obs_file_name)
        print('')

        print("Observing")
        virgo.observe(obs_parameters=obs, obs_file=obs_file_name)
        cal_file_name = "cal.dat"
        virgo.plot(obs_parameters=obs, n=20, m=35, f_rest=1420.4057517667e6,
                obs_file=obs_file_name, cal_file=cal_file_name,
                dB=True,  plot_file=prefix+'plot.png')
except Exception as e:
    print(e)
finally:
    pass
    # needed on windows to disable bias tee on the RTL-SDR
    # p = Popen('bias_tee_off.bat', cwd=r'./',shell=True)
    # stdout, stderr = p.communicate()
```

Run it every 5 minutes with cron and you have an automated backyard radio observatory!

```
crontab -e
*/5 * * * * ~/radio_astro/observe.py
```

# Wrapping Up 

I had a blast working on this project! It's amazing that with $200 in equipment, you can put together an observatory to detect the different arms of the Milky Way!

On my list of future upgrades is a nicer SDR, specifically the [USRP B200](https://www.ettus.com/all-products/ub200-kit/). I got to use one while working at GTRI and really enjoyed working with them. I'd also like to upgrade the post-observation portion of the observatory, automatically syncing data to the cloud and maybe even publishing live to a website.