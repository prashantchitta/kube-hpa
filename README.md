# kube-hpa
index.php page performs some CPU intensive computations.

###Step 1
First, we will start a deployment running the image and expose it as a service:
```
$ kubectl run php-apache --image=ecr.vip.ebayc3.com/kchitta/hpa-example --requests=cpu=200m --expose --port=80 --namespace=kchitta
service "php-apache" created
deployment "php-apache" created
```

###Step 2
Creating the autoscaler from a .yaml file
```
$ kubectl create -f hpa-php-apache.yaml
horizontalpodautoscaler "php-apache" created
```

We may check the current status of autoscaler by running:
```
$ kubectl get hpa --namespace=kchitta
NAME         REFERENCE                     TARGET    CURRENT   MINPODS   MAXPODS   AGE
php-apache   Deployment/php-apache/scale   30%       0%        1         10        18s
```

###Step 3
Increase the Load. We will start a container, and send an infinite loop of queries to the php-apache service (please run it in a different terminal):
```
$ kubectl run -i --tty load-generator --image=busybox /bin/sh --namespace=kchitta

Hit enter for command prompt

$ while true; do wget -q -O- http://php-apache; done
```

Within a minute or so, we should see the higher CPU load by executing:
```
$ kubectl get hpa --namespace=kchitta
NAME         REFERENCE                     TARGET    CURRENT   MINPODS   MAXPODS   AGE
php-apache   Deployment/php-apache/scale   30%       305%      1         10        3m
```

Here, CPU consumption has increased to 305% of the request. As a result, the deployment was resized to 7 replicas:
```
$ kubectl get deployment php-apache --namespace=kchitta
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   7         7         7            7           19m
```

###Step 4
Stop Load. In the terminal where we created the container with busybox image, terminate the load generation by typing <Ctrl> + C
Then we will verify the result state (after a minute or so):

```
$ kubectl get hpa --namespace=kchitta
NAME         REFERENCE                     TARGET    CURRENT   MINPODS   MAXPODS   AGE
php-apache   Deployment/php-apache/scale   30%       0%        1         10        11m

$ kubectl get deployment php-apache --namespace=kchitta
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   1         1         1            1           27m
```



#Autoscaling Algorithm

The autoscaler is implemented as a control loop. It periodically queries pods described by `Status.PodSelector` of Scale subresource, and collects their CPU utilization. Then, it compares the arithmetic mean of the pods' CPU utilization with the target defined in `Spec.CPUUtilization`, and adjusts the replicas of the Scale if needed to match the target (preserving condition: MinReplicas <= Replicas <= MaxReplicas).

The period of the autoscaler is controlled by the `--horizontal-pod-autoscaler-sync-period` flag of controller manager. The default value is 30 seconds.

CPU utilization is the recent CPU usage of a pod (average across the last 1 minute) divided by the CPU requested by the pod. In Kubernetes version 1.1, CPU usage is taken directly from Heapster.

The target number of pods is calculated from the following formula:

```TargetNumOfPods = ceil(sum(CurrentPodsCPUUtilization) / Target)```

Starting and stopping pods may introduce noise to the metric (for instance, starting may temporarily increase CPU). So, after each action, the autoscaler should wait some time for reliable data. Scale-up can only happen if there was no rescaling within the last 3 minutes. Scale-down will wait for 5 minutes from the last rescaling. Moreover any scaling will only be made if: `avg(CurrentPodsConsumption) / Target` drops below 0.9 or increases above 1.1 (10% tolerance). Such approach has two benefits:

* Autoscaler works in a conservative way. If new user load appears, it is important for us to rapidly increase the number of pods, so that user requests will not be rejected. Lowering the number of pods is not that urgent.

* Autoscaler avoids thrashing, i.e.: prevents rapid execution of conflicting decision if the load is not stable.
