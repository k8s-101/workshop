[Index](index) > The third party queue
======================================
_In this section, we'll deploy a queue-solution made by a third party, to our cluster. Then we'll use this queue to trigger speedtest-logger from speedtest-scheduler._

What queue? KubeMQ!
-------------------
TODO: How to get KubeMQ up and running on the cluster. Should probably end with looking at empty KubeMQ dashboard.

Publishing messages to the queue from speedtest-scheduler
---------------------------------------------------------
TODO: How to configure speedtest-scheduler to publish messages. Should probably end with looking at KubeMQ dashboard and noting it's not empty.

Configuring speedtest-logger to read messages from the queue
------------------------------------------------------------
TODO: All the stuff.

What now?
---------
Congratulations! You just completed the basic part of this workshop. Going forward we'll visit more advanced topics, starting with [Helm, the package manager for Kubernetes](5-helm-the-package-manager-for-kubernetes).