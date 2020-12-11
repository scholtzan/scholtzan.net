+++
title = "Speeding up Mozilla's automatic experiment analysis"
date = "2020-12-04"
type = "post"
tags = ["Mozilla", "Open Source", "Argo", "Python", "Jetstream", "Dask"]
+++

_This post describes my recent work on [jetstream](https://github.com/mozilla/jetstream) as part of my day job at Mozilla. In particular, I'll describe how [Argo](https://argoproj.github.io/) and [Dask](https://dask.org/) were used to scale up a system that processes dozens of experiments daily and on-demand. This work has been co-developed with my colleague [Tim Smith](https://github.com/tdsmith)._

Beginning of 2020 the development of a new experiment analysis infrastructure was launched at Mozilla which should help scaling up the number of experiments done and reduce the involvement of data scientists necessary for each experiment.
The entire infrastructure consists of different components and services, with [jetstream](https://github.com/mozilla/jetstream) being the component that automatically analyses collected telemetry data of clients enrolled in experiments. As the number of experiments started to increase, some of which require processing an extensive amount of data, running the daily automatic analyses started to take very long. In some cases completing the experiment analyses for a single day took over 23 hours. To provide analysis results without significant delay and speed up the entire analysis process, it was time to make some architectural changes and parallelize jetstream's experiment analysis. [Argo](https://argoproj.github.io/) looked perfect for parallelizing the analysis on a higher level and in combination with [Dask](https://dask.org/) for parallelizing lower-level calculations, we were able to significantly reduce the analysis runtime.

# Background: jetstream



![Jetstream Overview](/img/jetstream-overview.png)
*Jetstream Overview.*



* telemetry data Biguqyer
	* data docs

* overview picture
* analysis steps





* some general information about the experiment analysis infrastructure
	* Cirrus
	* some general information about Jetstream
		* experiment analysis steps
* Airflow
* jetstream-config

* problem
	* analysis taking too long
	* benchmarks
	* Airflow restrictions
* solution
	* setup
	* Argo
		* some general information
		* workflows
		* python client
		* benchmmarks
		* dashboard
	* Dask
		* some general information
		* adding Dask
			* rewriting without nesting
			* pickle problems
		* benchmakrs
* error handling?
	* bigquery logging
* pain points
	* permissions
* future work
	* dask kubernetes