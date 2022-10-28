

# Developing, Testing and Scaling Workflows




## Developing a New Workflow

### Start from a template
Preferably one that executes cleanly - even if it's Hello World.

### Chose Software Modules
Test interactively with software modules on the cluster to see what inputs are required, what parameters you want to specify, what outputs get created. 

### Add Tasks
Define tasks, using modules, test data (truncated files, scatters of 1), run and test.


### Scale Up Inputs
Start to run workflow on full size data (scatters of 1), then start to scatter over several files, then scatter over entire datasets. 


### Prep for Other Platforms and Sharing
Shift to docker containers instead of modules, ensure that all inputs are specified as files not directories!!, start to optmize compute resources required for tasks (how much memory does it really need, or is it going to need many CPU's).





## Approaches for Testing
- start small - start with truncated/downsampled data in the same formats but smaller!
- add tasks one at a time and leverage call caching to test single additions quickly
- start testing scatters with a scatter of one!


## Scaling Up, Moving to the Cloud
- Test with a scatter of many small files (not full size!)
- Begin optimizing computing resources for each task
- Dockerize single task environments
- Test locally on small data scatters of 1 to shift from modules to Dockerize
- Catch errors and adjust such as tools that rely on filesystems/directory structures which will break in the cloud
- Learn about the cloud you'll be using
  - what instance types exist and which are the best for you tasks/data
  - how do the data need to be stored in an object store?
  - how can you get permissions set up to access both the data and the compute resources?
