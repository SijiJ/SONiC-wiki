- [Overview](#overview)
    - [Scope](#scope)
    - [TestBed](#testbed)
- [Setup configuration](#setup-configuration)
    - [Ansible scripts to setup and run test](#ansible-scripts-to-setup-and-run-test)
        - [pfc_wd.yml](#pfc_wd.yml)
    - [Setup of DUT switch](#setup-of-dut-switch)
- [PFC Frames Generation](#pfc-frames-generation)
- [PTF Test](#ptf-test)
    - [Input files for PTF test](#input-files-for-ptf-test)
    - [Traffic validation in PTF](#traffic-validation-in-ptf)
- [Test cases](#test-cases)
- [Configuration Tests](#configuration-tests)
- [Functional Tests](#functional-tests)
- [Open Questions](#open-questions)

## Overview
The purpose is to test the functionality of PFC Watchdog on the SONIC switch DUT
The test assumes all necessary configuration, PFC Watchdog, and BGP routes, are already pre-configured on the SONIC switch before test runs.

### Scope
The test is targeting a running SONiC system with fully functioning configuration.
Purpose of the test is to verify a SONiC switch system correctly detecting PFC storm, reacting on it according to the configuration.

### Testbed 
T0, T0-112, T0-64, T1, T1-lag

## Setup configuration
PFCWD configuration will be set on DUT dynamically.
### Ansible scripts to setup and run test
#### pfc_wd.yml

pfc_wd.yml when run with testname "pfc_wd" will do the following:

1. Get the metadata of the ports to be tested
2. Generate and apply PFCWD configuration.
3. Run tests:
4. Clean up dynamic configuration and temporary configuration on the DUT.

PFC WD test consists of a number of subtests, and each of them will include the following steps, using loganalyzer to determine whether expected log exists in syslog:
1. Run lognanalyzer 'init' phase
2. Run PFC WD Sub Test
3. Run loganalyzer 'analyze' phase

Subtests will test either a configuration or watchdog functionality.

#### Setup of DUT switch
Setup of SONIC DUT will be done by Ansible script. During setup, Ansible will copy JSON file containing configuration for Watchdog to DUT and pushed to SONiC config DB via sonic-cfggen. Data will be consumed by orchagent.

JSON Sample:

pfc_wd.json
```
{
    "PFC_WD_TABLE": {
        "Ethernet4": {
            "action": "drop",
            "detection_time": "500",
            "restoration_time": "5000"
       }
    }
}
```

## PFC Frames Generation

PFC frames are generated directly from fanout switches using script pfc_gen.py

In order to verify both actions (drop/forward), traffic will be sent through a stormed port and expected to be forwarded or dropped respectively.

##Test cases

Each test case will be additionally validated by the loganalizer utility.

Some cases will add PFC WD configuration at the beginning and remove them at the end. They are grouped into two categories: configuration tests and traffic tests.

### Configuration Tests

### Test case \#1 - Configure forward action

#### Test objective

Verify that watchdog with forwarding action will be successfully configured and then removed from the specified port.

#### Test steps

- Initialize log analyzer.
- Apply JSON file with WD configuration with forwarding action.
- Verify that log has the message saying that WD started on specified port.
- Initialize log analyzer.
- Apply JSON file with removal of WD configuration with forwarding action.
- Verify that log has the message saying that WD stopped on specified port.

### Test case \#2 - Invalid action

#### Test objective

Verify orchagent correctly handles invalid WD action field.

- Initialize log analyzer.
- Apply JSON file with WD action other than "drop"/"forward".
- Verify error in logs saying "Invalid PFC WD action".

### Test case \#3 - Invalid detection time

#### Test objective

Verify orchagent correctly handles invalid "detection_time" field.

- Initialize log analyzer.
- Apply JSON file with WD mitigation_time other than unsigned int.
- Verify error in logs saying "Invalid PFC WD action".

### Test case \#4 - Invalid restoration time

#### Test objective

Verify orchagent correctly handles invalid "restoration_time" field.

- Initialize log analyzer.
- Apply JSON file with WD restoration_time other than unsigned int.
- Verify error in logs saying "Invalid PFC WD action".

### Functional Tests

### Test case \#5 - Trigger forward action when queue buffer is empty

#### Test objective

Verify that WD is triggered under PFC storm and TX/RX packets are forwarded from the interface.

- Initialize log analyzer.
- Apply JSON file with WD configuration with forwarding action.
- Verify that log has the message saying that WD started on specified port.
- Initiate PFC storm on the specified port.
- Verify that logs contain the message that PFC WD was triggered on specified port/queue.
- Start sending traffic to the specified port.
- Verify that traffic is forwarded.
- Start sending traffic from the specified port.
- Verify that traffic is forwarded.
- Stop PFC storm.
- Verify that logs contain the message that queue was restored from the storm.
- Remove PFC WD configuration from the port.

### Test case \#6 - Trigger drop action when queue buffer is empty

#### Test objective

Verify that WD is triggered under PFC storm and packets are drop on the interface.

- Initialize log analyzer.
- Apply JSON file with WD configuration with the drop action.
- Verify that log has the message saying that WD started on specified port.
- Initiate PFC storm on the specified port.
- Verify that logs contain the message that PFC WD was triggered on specified port/queue.
- Start sending traffic to the specified port.
- Verify that traffic is dropped.
- Start sending traffic from the specified port.
- Verify that traffic is dropped.
- Stop PFC storm.
- Verify that logs contain the message that queue was restored from the storm.
- Remove PFC WD configuration from the port.

### Test case \#7 - Trigger forward action when queue buffer is not empty

#### Test objective

Verify that WD is triggered under PFC storm and packets are forwarded from the interface.
- Initiate PFC storm on the specified port.
- Send data packet so that the queue buffer will not be empty
- Initialize log analyzer.
- Apply JSON file with WD configuration with forwarding action.
- Verify that log has the message saying that WD started on specified port.
- Verify that logs contain the message that PFC WD was triggered on specified port/queue.
- Start sending traffic to the specified port.
- Verify that traffic is forwarded.
- Start sending traffic from the specified port.
- Verify that traffic is forwarded.
- Stop PFC storm.
- Verify that logs contain the message that queue was restored from the storm.
- Remove PFC WD configuration from the port.

### Test case \#8 - Trigger drop action when queue buffer is not empty

#### Test objective

Verify that WD is triggered under PFC storm and TX/RX packets are dropped on the interface.
- Initiate PFC storm on the specified port.
- Send data packet so that the queue buffer will not be empty
- Initialize log analyzer.
- Apply JSON file with WD configuration with the drop action.
- Verify that log has the message saying that WD started on specified port.
- Verify that logs contain the message that PFC WD was triggered on specified port/queue.
- Start sending traffic to the specified port.
- Verify that traffic is dropped.
- Start sending traffic from the specified port.
- Verify that traffic is dropped.
- Stop PFC storm.
- Verify that logs contain the message that queue was restored from the storm.
- Remove PFC WD configuration from the port.


### Test case \#9 - Timers accuracy

#### Test objective

Verify that WD detects storm and restores queue within specified time constraints, run 20 times and verify the median.

- Initialize log analyzer.
- Apply JSON file with WD configuration.
- Initiate PFC storm on the specified port.
- Stop PFC storm.
- Verify that WD was triggered within "mitigation_time".
- Verify that queue was restored after "restoration_time".
- Remove PFC WD configuration from the port.

### Test case \#10 - Stress/Resources tests

#### Test objective

Verify that WD is able to mitigate storm on all lossless queues of all ports

- Initialize log analyzer.
- Apply JSON file with WD configuration.
- Simulate PFC storm on all queues and ports using DEBUG_STORM field in DB.
- Verify that WD was triggered on all queues.
- Stop PFC storm.
- Remove PFC WD configuration from the port.

Test must be performed for both drop and forward actions. For all the tests, PFCWD is enabled on all ports