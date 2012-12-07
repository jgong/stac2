# Stac2

Stac2 is a load generation system for Cloud Foundry. The system is made up for several Cloud Foundry applications. The instance
count of some of the applications gates the amaount of load that can be generated and depending on the size and complexity of your system,
you will have to size these to your Cloud Foundry instance using the "vmc scale --instances" command.


Installing/running Stac2 on your Cloud Foundry instance requires a small amount of configuration and setup.

* clone the stac2 repo "git clone git@github.com:cloudfoundry/stac2.git"
* create a cloud config file for your cloud into nabv/config/clouds
* create user accounts used by stac2 using "vmc register"
* note, you must use vmc version 0.4.2 or higher.
* from the stac2 repo root, where the manifest.yml file is run vmc push (note, you must have vmc 4.2 or higher, cfoundry 0.4.6 or higher, and manifests-vmc-plugin 0.4.17 or higher for the push to succeed)
* based on the size of your cloud and desired concurrency you will need to adjust the instance counts of nabv, nabh, and nf
    * nabv should be sized to closely match the concurrency setting in your cloud config. It should be a little over half your desired concurrecny.
    for a large production cloud with a cmax of 192 set the instance count of nabv to ~100 (vmc scale nabv --instances 100)
    * nabh should be between 16 and 32, depending on how hot of an http load you plan to run, at 32 you can easily overwhelm a small router pool with so a rule of thumb would be to start with 16 for must mid-sized (100+ dea clouds), 32 for 250+ dea clouds, and4-8 for very small clouds
    * nf should be set large IFF you plan on running the high thruput xnf_ loads. Tun run 30,000 http requests per second run between 75 and 100 instances of nf
    * stac2 is a single instance app so leave it at 1

Once stac2 is running and configured correctly, since this is the first time its been run, you need to populate it with some workloads. The static
starter set of workloads are in stac2/config/workloads. The workload selection UI has an "edit" link next to the selector so to initially
populate the default workloads click the "edit" link then the "reset workloads" link, then return the the main screen by clicking
on the "main" link. Note, any time you want to edit/reload the default workloads, just edit the file under stac2/config/workloads, re-push stac2,
and then go through the edit/reset workloads/main cycle. Fine approach for the default workloads. If you just want to change/add a workload you can do this by just uploading a new workload file
on the edit page, or delete and reload an existing workload file. The edit/reset/main path is really just for resetting things to the default state. Apologies for the
ugly edit form. What can I say. Lazy hack session one night when I just wanted to be able to let our ops guys edit/upload a workload so I built a form in an hour.

At this point, assuming your cloud config file is correct, you should be able to run some load. Select the sys_info workload in the workload selection, validate
that the cloud selector correctly selected your cloud, and then click the light gray 100% load button. You should immeadiately see the blue load lights
across the bottom of the screen peg to your max concurrency and you should see the counters in the main screen for the info call show activity. For a reasonable
cloud with reasonable stac2 concurrency seeing ~1,000 CC API calls per second (the yellow counter) should be easily in reach. You should see a screen similar to this:

<p/>
![Stac2 in Action](https://github.com/cloudfoundry/stac2/raw/master/images/stac2_home.png)
<p/>

If you see no activty, then click on the reset button above the load light grid and try again. Worst case, restart nabh, then nabv,
hit the reset button, and then restart.



# Stac2 Display and Controls

Stac2 has several output only regions and only a small set of input controls.

* **timeclock** - this is a counter in the upper right that starts counting up once a scenario is started, and stos when the reset button is pressed
* **on/off controls** - in the upper left of the screen there are 6 gray'd out buttons. The first is an on/off light. When this is red, it means a scenario is running and the scenario can be
drained by clicking the red light. Next there are 5 load buttons labeled low, 25%, 50%, 75%, 100%. These determine the relative load thats going to be launched at the cloud.
Start a load by clicing one of these buttons and increase/decrease by clicking various levels. Note, since scenarios take time, decreasing the load or turning the
system off is not instantaneous. The system drains itself of work naturally and depending on the mix this could take seconds to minutes. Stac2 is inactive when the
on/off control is light gray and none of the blue lights are on in the light grid at the bottom of the screen. Make sure to let stac2 go idle before restarting componets, etc.
* **load selector** - the load selector is below the light grid and is used to select the scenario that you intend to run. If this control is empty, follow the edit/reset/main process outlined above.
* **cloud selector** - leave this alone
* **http request counter** - this green counter on the upper right shows the amount of http traffic sent to apps created by stac2 workloads, or in the case of static loads like xnf_*, traffic to existing apps.
* **cc api calls** - this yellow counter shows the number of cloud foundry api calls executed per second
* **results table** - this large tabluar region in the center of the screen shows the various api calls used by their scenarios displacyed as ~equivalent vmc commands.
    * *total* - this column is the raw number of operations for this api row
    * *<50ms* - this column shows the % of api calls that executed in less than 50ms
    * *<100ms* - this column shows the % of api calls that executed in less than 100ms but more than 50ms. The column header misleads you to believe that this should include all of the <50% calls as well, but thats not the intent. Read this column as >50ms & <= 100ms.
    * *<200ms, <400ms,...>3s* - these work as you'd expect per the above definitions
    * *avg(ms)* - the average speed of the calls executed in this row
    * *err* - the % of api calls for this row that failed, note, failures are displayed as a running log under the light grid. On ccng based cloud controllers the host IP in the display is a hyperlink that takes you to a
    detail page showing all api calls and request id's that occured during the mix that failed. The request id can be used for log correleation while debugging the failure.
* **results table http-request row** - this last row in the main table is similar to the previous description, but here the "api" is any http request to an app created by a scenario or a static app in the case of xnf based loads
* **results table http-status row** - this row shows the breakdown of http response status (200's, 400's, 500's) as well as those that timed out
* **email button** - if email is enabled in your cloud config, this button will serialize the results table and error log and send an email summary. Note, for non-email enabled clouds, the stac2 front end entrypoing "/ss2?cloud=cloud-name" will produce a full JSON dump as well.
* **dirty users?** - great name... If you hit reset during an active run, or bounced your cloud under heavy activity, or restarted/repushed nabv or nabh during heavy activity, you likely left the system in a state where there
are applications, services, routes, etc. that have not been properly cleared. If you see quota errors, or blue lights stuck on, thats another clue. Use this button on an idle system to ask stac2 to clean up these zombied apps and services.
Under normal operation you will not need to use this as the system self cleans anywhere it can.
* **reset** - this button flushes the redis instance at the center of the system wiping all stats, queues, error logs, etc. always use between runs on a fully quiet system. if you click on this where there is heavy activity, you will
more than likley strand an application or two. If your system is serioulsy hosed then click reset, then vmc restart nabh; vmc restart nabv; then click reset again. This very hard reset yanks redis and restarts all workers.
* **light grid** - each light, when on, represents an active worker in the nabv app currently running an instance of the selected load. For instance if a load is designed to simulate login, push, create-service, bind-service, delete, delete-service, if one worker is currently running
the load, one light will be on. If 100 lights are on, then 100 workers are simultaneously executing the load. Since stac2 is designed to be able to mimic what a typical developer is doing in front of the system, you can think if the lights as representing
how many simultaneously active users the system is servicing. Active means really active though so 100 active active users can easily mean 10,000 normal users.

# Components

Stac2 consists of several, statically defined Cloud Foundry applications and services. The [manifest.yml](https://github.com/cloudfoundry/stac2/blob/master/manifest.yml) is used by the master vmc push
command to create and update a stac2 cluster. The list below describes each of the static components as well as their relationship to one and other.

## stac2

The stac2 application is responsible for presenting the user interface of stac2. It's a simple, single-instance ruby/sinatra app. Multiple browser sessions may have this app open and each sessions will see
the same data and control a single instance of a stac2 cluster. The bulk of the UI is written in JS which the page layout and template done in haml. When a stac2 run is in progress a lot of data is generated
and the UI is supposed to feel like a realtime dashboard. As a result, there is very active JS based polling going on (10x/s) so its best have only a small number of browser sessions open at a time.

One key entrypoint exposed by stac2 is the ability to capture all of the data rendered by the UI in it's raw JSON form (/ss2?cloud=name-of-cloud).

The stac2 app is bound to the stac2-redis redis service-instance as well as the stac2-mongo mongodb service-instance.

## nabv

The application is the main work horse of the system for executing VMC style commands (a.k.a., Cloud Controller API calls). Note: The nab* prefix came from an earlier project around auto-scaling where
what I needed was an apache bench like app that I could drive programatically. I built a node.JS app called nab (network ab) and this prefix stuck as I morphed the code into two components...

The nabv app is a ruby/sinatra app that makes heavy use of the cfoundry gem/object model to make synchronous calls into cloud foundry. It's because of the synchronous nature of cfoundry that this app
has so many instances. It is multi-threaded, so each instance drives more than one cfoundry client, but at this point given ruby's simplistic threading system nabv does not tax this at all...

The app recieves all of its work via a set of work queue lists in the stac2-redis service-instance. Each worker does a blpop for a work item, and each work item represents scenario that the worker is supposed to run. The scenarios
are described below but in a nutshell they are sets of commands/cloud controller APIs and http calls that should be made in order to simulate developer activity.

## nabh

## nf

## stac2-redis

## stac2-mongodb

# Workloads

# Clouds

# Trivia

Where did the name stac2 come from? The Cloud Foundry project started with a codename of b20nine. This code name was inspired by Robert Scoble's [building43](http://www.building43.com/) a.k.a., a place where
all the cool stuff at Google happened. The b20nine moniker was a mythical place on the Microsoft Campus (NT was mostly built in building 27, 2 away from b20nine)... Somewhere along the way, the b20nine long form
was shortened to B29, which unfortunately was the name of a devestating machine of war. In an effort to help prevent Paul Maritz from making an embarassing joke using the B29 code name, a generic codename of "appcloud"
was used briefly.

The original STAC system was developed by Peter Kukol during the "appcloud" era and in his words: "My current working acronym for the load testing harness is “STAC” which stands for the (hopefully) obvious
name (“Stress-Testing of/for AppCloud”); I think that this is a bit better than ACLH, which doesn’t quite roll off the keyboard. If you can think of a better name / acronym, though, I’m certainly all ears."

In March 2012, I decided to revisit the load generation framework. Like any good software engineer, I looked Peter's original work and declared it "less than optimal" (a.k.a., a disaster) and decided that the only way to "fix it" was to do a complete and total re-write. This work was done
during a 3 day non-stop coding session where I watched the sunrise twice before deciding I'm too old for this... The name Stac2 represents the second shot at building a system for "Stress-Testing of/for AppCloud". I suppose
I could have just as easily named it STCF but given rampant dyslexia in the developer community I was worried about typos like STFU, etc... So I just did the boring thing and named it Stac2. Given that
both Peter and I cut our teeth up north in Redmond, I'm confident that there will be a 3rd try coming. Maybe that's a good time for a new name...