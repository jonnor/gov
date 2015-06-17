guv
===

`guv`, aka Grid Utilization Virgilante, is a governor for your (Heroku) workers:
It automatically scales the numbers of workers based on number of pending jobs in a (RabbitMQ) queue.

The number of workers is calculated to attempt that all jobs are completed within a specified *deadline*.
This is based on estimates of the processing time (mean, variance).

guv is written in Node.js, but can be used with workers in any programming language.

[Origin of the word guv](http://english.stackexchange.com/questions/14370/what-is-the-origin-of-the-british-guv-is-it-still-used-colloquially).

## Status

*Experimental*

* Supports RabbitMQ messaging system and Heroku workers
* Simple algorithm for scaling to hit a deadline exists
* Scaling algorithm does not compensate for worker boot time
* Used in production at [The Grid](https://thegrid.io), with [MsgFlo](https://github.com/msgflo/msgflo)
* ! Missing tests

## Usage

Install as NPM dependency

    npm install --save guv
    
Add it to your Procfile

    echo "guv: node node_modules/.bin/guv" >> Procfile

Configure an Heroku API key to use. Get it 

    heroku config:set HEROKU_API_KEY=`heroku auth:token`

Configure RabbitMQ instance to use. It must have the management plugin installed and configured.

    heroku config:set GUV_BROKER=amqp://[user:pass]@example.net/instance

Note: If you use CloudAMQP, guv will automatically respect the `CLOUDAMQP_URL` envvar. No config needed.

For guv own configuration we also recommend using an envvar.
This allows you to change the configuration without redeploying.
See below for details on the configuration format.

    heroku config:set GUV_CONFIG="`cat autoscale.guv`"

To verify that guv is running and working, check its log.

    heroku logs --app myapp --ps guv


## Configuration

One guv instance can handle multiple worker *roles*.
Each role has an associated queue, worker and scaling configuration - specified as variables.

    # comment
    myrole{variable1=value1}
    otherrole{variable2=value2}

The special role name `*` is used for global, application-wide settings.
Each of the individual roles will inherit this configuration if they do not override it.

    # Heroku app is my-heroku-app, defaults to using a minimum of 5 workers, maximum of 50
    *{min=5,max=50,app=my-heroku-app}
    # uses only defaults
    imageprocessing{}
    # except for text processing
    textprocessing{max=10}

guv attempts to scale workers to be within a `deadline`, based on estimates of `processing` time.
To let it do a good job you should always specify the deadline, and *mean* processing time.

    # times are in seconds
    textprocessing{deadline=100, processing=30}

You can also specify the variance, as 1 standard deviation

    # 68% of jobs complete within +- 3 seconds
    textprocessing{deadline=100, processing=30, stddev=3}

The name of the `worker` and `queue` defaults to the `role name`, but can be overridden.

    # will use worker=colorextract and queue=colorextract
    colorextract{}
    # explicitly specifying
    histogram{queue=hist.INPUT, worker=processhistograms}

For list of all supported configuration variables see [./src/config.coffee](./src/config.coffee).
Many of the commonly used ones have short and long-form names.

You can put multiple role definitions on single line by delimiting with `;`

    first{foo=bar}; second{faz=baz}

guv configuration files by convention use the extension *.guv*, for instance `autoscale.guv` or `myproject.guv`.

# Best practices

* Measure the actual processing times of your jobs.
Calculate mean and standard deviations, and use these in the `guv` config.

* Measure the actual end-to-end processing time for your jobs.
Monitor that they meet your target deadlines, and identify the load cases where they do not.

* Keep boot times of your workers as low as possible.
Responsiveness to variable loads, especially sudden peaks is severely affected by boot time.
Avoid doing expensive computations, lots of I/O or waiting for external services.
If neccesary do more during app build time, like creating caches or single-file builds.

* Separate out jobs with different processing time characterstics to different worker roles
If your job processing time has several clear peaks instead of one in the processing time histogram,
determine what the different cases are and use dedicated worker roles.
If your job processing time depends on the size of the job payload, implement an estimation
algorithm and route jobs into N different queues depending on which bin they fall into.

* Separate out jobs with different deadlines to different workers
Maintenance work, and anything else not required to maintain responsiveness for users,
should be done in separate queues, usually with a higher deadline.
This ensures that such work does not disrupt quality-of-service.

* Use only one primary input queue per worker

