[TOC]

# Notes for System Administrators

This page is for all the sysadmins and devops folks tasked with operationalizing Adhearsion applications.

## Interacting with the Adhearsion Process

Starting with Adhearsion 2.0, all Adhearsion applications will appear in your process list with the name "ahn" and that name should work with standard tools like ps/pkill/killall.

## Signals

Adhearsion 2.0 processes respond to several signals:

* On SIGALRM, Adhearsion will toggle on or off TRACE level logging.  This can be very helpful when debugging a production system.
* On SIGINT or SIGQUIT Adhearsion does several things:
    - On first signal, Adhearsion marks its internal state as "shutting down" but continues to take and process calls normally.
    - On second signal, Adhearsion will continue to process calls already in the system, but will reject any new calls.
    - On the third signal, Adhearsion will send a HANGUP to all existing calls.
    - In all of the above cases Adhearsion will shut down as soon as the call count reaches 0.
    - On the fourth signal, Adhearsion will stop immediately (forced shutdown).

## Starting and Stopping Adhearsion Processes

There are two ways to start an Adhearsion process:

* In the foreground with an Adhearsion console.  This is the common case while developing Adhearsion applications.  To start in this mode call ahn with the "start" argument: "ahn start /path/to/my/application"

* In the background as a daemon.  This is typical for deployment scenarios.  To start in this mode call ahn with the "daemon" argument: "ahn daemon /path/to/my/application"

<div class='docs-progress-nav'>
  <span class='back'>
    Back to <a href="/docs/best-practices/deployment">Deployment</a>
  </span>
</div>