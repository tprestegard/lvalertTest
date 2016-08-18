# lvalertTest

# FUNCTIONALITY

The idea is to set up a modular library which allows us to simulate data corresponding to all stages of an event's life in GraceDb. Each bit of data is generated by a different class within several modules. These classes all must have "genSchedule" methods, which generate a schedule of Actions as defined in ~/lib/schedule.py. Then, all these schedules are stitched together (Schedules know how to add themselves together) and we have an iterable list of things (with associated wait times) that we should do to simulate a full event.

Note, simulating multiple events can be sewn together by simply adding their Schedules.

Also, we may not know which GraceID will be assigned to a particular event a priori (ie: at instantiation time). Therefore, we provide a simple class that can be shared between actions associated with a single event. The class contains protected attributes that must be accessed through getters and setters. Thus, when an entry is actually created in GraceDB, the object can be updated and the information will be available to all associated actions in the future.

We also create a environment which mimics interactions with remote GraceDb and LVAlert servers but only uses local filesystem queries. This allows developers to thoroughly test their code before ever connecting it to a real server (which might ping people if something breaks). By combining the offline tools with the simulation suite, large scale tests can be carried out quickly and locally without impacting others. In particular, the LVALert tools come in several different flavors, some of which publish data to local files and some of which monitor those local files to distribute alerts either over the actual LVAlert servers or to other local processes.

--------------------------------------------------

# EXECUTABLES

 - ~bin/simulate.py 
   - generates fake data corresponding to events from pipelines as well as their expected follow-up and submits them to either GraceDb or FakeDb (see LIBRARIES:simulation and LIBRARIES:FakeDb).
 - ~bin/lvalertTest_commandMP
   - generates lvalertMP command messages but writes them into a file so the LVAlertTest infrastructure can identify and distribute them. Requires the LVAlertMP libraires.
 - ~bin/lvalertTest_listen
   - a script that mimics the behavior of lvalert_listen but looks at local files (see LIBRARIES:LVALertTest) rather than a pubsub node.
 - ~bin/lvalertTest_listenMP
   - a script that mimics the behavior of lvalert_listenMP but looks at local files (see LIBRARIES:LVAlertTest) rather than a pubsub node.
 - ~bin/lvalertTest_overseer
   - a script that looks at local files (see LIBRARIES:LVAlertTest) and then publishes the lvalert messages discovered therein to a bone fide LVAlert server. 
 - ~bin/lvalertTest_replay
   - a script that queries GraceDb or FakeDb (see LIBRARIES:FakeDb) and then generates simulated LVAlert messages corresponding to event creation and the full log of that event. The messages are written into a local file (see LIBRARIES:LVAlertTest) and can then be distributed with lvalertTest_listen, lvalertTest_listenMP, or lvalertTest_overseer. Note: this allows users to reproduce *exactly* the same series of messages, spaced in time the same way, repeatedly and as many times as they like.

We note that there are also a few ancilliary executables included (~bin/confirmation.sh, ~bin/sanityCheck_FakeDb.py, ~bin/checkPermissions.py, ~bin/lvalertMP_test.py) which are included for internal tests but are not really likely to be useful to the user.

--------------------------------------------------

# LIBRARIES
 
There are 3 main libraries

  - simulation of events and follow-up processes
  - FakeDb (mimics GraceDb rest interface)
  - LVAlertTest utilities (provides distribution of messages from FakeDb)

-----------
simulation:

The simulation is encapsulated in ~/lib/simUtils.py, ~/lib/pipelines.py, ~/lib/pe.py, ~/lib/dq.py, and ~/lib.schedule.py. The basic operations are provided in schedule.py, which declares a Schedule class (essentially an ordered list), which contains Actions (wrappers around function calls with a .execute() method). For simulations, all the actions boild down to interactions with GraceDb (or FakeDb) through a REST interface and these are defined within schedule as CreateEvent, WriteLog, WriteFile, and WriteLabel. Thus, we construct a set of Actions, add them into a Schedule, and then iterate over the Schedule to perform each Action.

Given a config file for a type of event (several examples are provided in ~/etc), simUtils.genSchedule will determine which follow-up processes are to be simulated. It delegates to objects defined within pipelines.py, dq.py, and pe.py to create Schedules of different parts of the simulation (each object has a genSchedule method). All these little Schedules are added together, with appropriate parent/child relationships if needed (eg: Bayestar only starts after psd.xml.gz has been uploaded), to generate a complete Schedule for the entire event. Several events can be simulated simultaneously by adding their individual Schedules together as well. We then iterate over the global Schedule and perform each Action when the time is right.

We note that several config files can be provided, and events will be simulated by randomly choosing one config file at a time.

-----------
FakeDb

FakeDb (~/lib/ligoTest/gracedb/rest.py) dummies up most of the interactions provided by the GraceDb REST interface, but manages data locally through a specific directory structure. It also returns FakeTTPResponses and raises FakeTTPErrors as needed. In particular, it generates responses to queries (for everthing exept GraceDb.events) that should be indistinguishable from their counterparts from GraceDb. 

FakeDb also formats LVAlert messages corresponding to createEvent, writeLog, writeFile, and writeLabel calls and writes them to a local file with the corresponding node. This file can be monitored by the LVAlertTest tools to distribute the messages as needed.

-----------
LVAlertTest

LVAlertTest (~/lib/ligoTest/lvalert/lvalertTestUtils.py) provides basic wrappers to monitor files and parse nodes and messages from them. This is used within FakeDb to structure the LVAlert messages it produces. Furthermore, this module provides classes that can act as daemon monitors, quickly detecting and distributing new alert messages as they come in. This is done primarily thorough the LVAlertBuffer class, which delegates to the FileMonitor class, both declared within lvalertTestUtils.py. The module also contains a few helper functions that control how we distribute events (one for each executable).

--------------------------------------------------

# EXAMPLES

Here is an example command line that simulates 5 cWB events, submitting them to FakeDb, and listening to them with lvalertTest_listen. All the required files for this example live in the repository, although you may have to modify some config files to point to the correct executables (depending on where you clone the repo)

>  git clone https://github.com/reedessick/lvalertTest.git

>  cd lvalertTest

>  . setup.sh

>  mkdir -p test/FakeDb

>  touch test/FakeDb/lvalert.out

>  bin/lvalertTest_listen -f test/FakeDb -c etc/lvalert_listen.ini 1> test/lvalertTest_listen.out 2> test/lvalertTest_listen.err &

>  bin/simulate.py -g test/FakeDb -N 5 -i H1,L1 -o test/ etc/cwb.ini

Note: we touch test/FakeDb/lvalert.out because lvalertTest_listen expects it to exist. We then launch lvalertTest_listen in the background. At this point, we run simulation.py to generate the 5 cWB events distributed uniformly through time at a rate of 0.1 Hz. simulate.py provides several command-line options to adjust these parameters, including changing the distribution from uniform to poisson and specifying durations of the experiment rather than a fixed number of events.

--------------------------------------------------
--------------------------------------------------

Note, there is a bone fide lvalert node specifically for testing purposes. Here are the pertinents:

    username : gdb_processor
    password : ***

    node : lvalerttest-testnode

I've also written a simple script to ping this node with basic messages

    ~/bin/lvalertMP_test

and a config file with which I can run lvalert_listenMP

    ~/etc/lvalert_listenMP.ini (points to ~/etc/childConfig.ini)

as well as a config file for a more standard lvalert_listen process to confirm we heard the alerts.

    ~/etc/lvalert_listen.ini (delegates to ~/bin/confirmation.sh)

This provides a simple platform from which we can test things. However, using lvalert_commandMP is also an option.
