+++
title = "Speeding up Mozilla's automatic experiment analysis"
date = "2020-12-04"
type = "post"
tags = ["Mozilla", "Open Source", "Argo", "Python", "Jetstream", "Dask"]
+++

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