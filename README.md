# cronj

This is *another* cron-inspired task-scheduling library. I have found many scheduling libraries for clojure:

  - [cron4j](http://www.sauronsoftware.it/projects/cron4j)
  - [clj-cronlike](https://github.com/kognate/clj-cronlike)
  - [at-at](https://github.com/overtone/at-at)
  - [monotony](https://github.com/aredington/monotony)
  - [quartzite](https://github.com/michaelklishin/quartzite)

The first two follow the cron convention. The "task" (also called a "job") can only be scheduled at whole minute intervals. [at-at](https://github.com/overtone/at-at) has milli-second resolution, but was limited in the number of threads that have to be predetermined. It was good for looking after tasks that did not overlap between calls but not for tasks that may take an arbitarily long time. [monotony](https://github.com/aredington/monotony) uses core.logic, which is something that I am yet to understand. If I had known [quartzite](https://github.com/michaelklishin/quartzite) had a field allowing seconds, I may have just used that but alas, I only found out when the library was announced on [google groups](https://groups.google.com/forum/?fromgroups=#!topic/clojure/SN4DdqFKIh4). I guess my only comment to the difference is that quarzite has more features but cronj has less dependencies and is written in pure clojure (relying only on clj-time, which is more or less a clojure staple for any sort of time-based application).

I needed something that

  - started scheduled tasks with a per-second interval having high system-time accuracy without wasting system resourcs.
  - would spawn as many threads as needed, so that tasks started at earlier intervals could exist along side tasks started at later intervals.
  - an additional design requirement required that task handlers are passed a date-time object, so that the handler itself is aware of the time when it was initiated. (something that quartz does not explicitly support)

### Installation:

In project.clj, add to dependencies:

     [cronj "0.5.2"]


### New Features:

#### v0.5.2

This version shifted various components around.

There is now a `cronj.global` to put the global timesheet and timeloop datastructures. Tasks are now triggered a watch function, as opposed to being triggered in the time-loop. I find this to be much more elegant and leads to looser coupling between the components, allowing for better component testing, which I still need to add.

I have moved `cronj.task`, `cronj.tab` and `cronj.timesheet` to  `cronj.data.task`, `cronj.data.tab` and `cronj.data.timesheet` respectively. `cronj.timekeeper` has been renamed to `cronj.loop`.

The `cronj.core` namespace has remained pretty constant. The `new-cronj!!` and `set-cronj!!` methods have been taken out, mainly to stop users (myself mainly) from shooting themselves in the foot by creating lots of cronj instances. Previously, there could be multiple cronj loops running with multiple timesheets. Now there now can only be one cronj time-loop initiated.


### Usage

    (require '[cronj.core :as cj])

    (cj/load-tasks!
     [{:id 1 :desc 1 :handler #(println "Task 1: " %) :tab "0-60/2 * * * * * *"}
      {:id 2 :desc 2 :handler #(println "Task 2: " %) :tab "1-60/2 * * * * * *"}])

    (cj/start!)

    ;; wait for outputs ......
    ;;> Task 1:  #DateTime 2012-09-17T14:10:24.000+10:00
    ;;> Task 2:  #DateTime 2012-09-17T14:10:25.001+10:00
    ;;> Task 1:  #DateTime 2012-09-17T14:10:26.000+10:00
    ;;> Task 2:  #DateTime 2012-09-17T14:10:27.001+10:00
    ;;> Task 1:  #DateTime 2012-09-17T14:10:28.000+10:00

    ;; get bored and stop
    (cj/stop!)


### Arguments
Additional arguments can be added to the handler through the args option and a function that is takes the arguments

    (cj/unschedule-all-tasks!)
    (cj/schedule-task! {:id 3 :desc 3
                        :handler (fn [dt & {:keys [task-name]}] (println task-name ": " dt))
                        :tab "/2 * * * * * *"
                        :args {:task-name "Hello There"}})
    (cj/start!)
    ;; more outputs ......
    ;;> Hello There :  #DateTime 2012-09-17T14:53:10.000+10:00
    ;;> Hello There :  #DateTime 2012-09-17T14:53:12.000+10:00
    ;;> Hello There :  #DateTime 2012-09-17T14:53:14.001+10:00

    ;; stop outputs
    (cj/stop!)


### More Examples

    (cj/schedule-task! {:id "print-date-1"
             :desc "prints out the date every 5 seconds"
             :handler #'println
             :tab "/5 * * * * * *"})

    (cj/schedule-task! {:id "print-date-2"
            :desc "prints out the date every 5 seconds between 32 and 60 seconds"
            :handler #'println
            :tab "32-60/5 * * * * * *"})

    (cj/schedule-task! {:id "print-date-3"
             :desc "prints out the date every 5 seconds on the
                    9th aand 10th minute of every hour on every Friday
                    from June to August between the year 2012 to 2020"
             :handler #'println
             :tab "/5  9,10  * 5 * 6-8 2012-2020"})

### Hooks
For additional control like cleanup and other side-effecting operations, post and pre hooks can also be set.

    (cj/unschedule-all-tasks!)
    (cj/schedule-task!
    {:id        "hook-example"
     :desc      "This is showing how a hook example should work"
     :handler   (fn [dt & args]
                   (println "Instance:" dt " - In Handle, Arguments:" args)
                   (Thread/sleep 1000) ;; Do Something
                   "Result")
     :pre-hook  (fn [dt args]
                   (println "Instance:" dt " - In Pre-Hook," "Arguments:" args)
                   (assoc args :pre "Pre-Hooked"))
     :post-hook (fn [dt args]
                   (println  "Instance:" dt " - In Post-Hook, Arguments: " args))
     :args      {:a "a" :b "b"}
     :tab       "/5 * * * * * *"})

    (cj/start!)

    ;;> Instance: #<DateTime 2012-09-18T10:00:15.001+10:00>  - In Pre-Hook, Arguments: {:a a, :b b}
    ;;> Instance: #<DateTime 2012-09-18T10:00:15.001+10:00>  - In Handle, Arguments: (:pre Pre-Hooked :a a :b b)
    ;;> Instance: #<DateTime 2012-09-18T10:00:15.001+10:00>  - In Post-Hook, Arguments:  {:result Result, :pre Pre-Hooked, :a a, :b b}
    ;;> Instance: #<DateTime 2012-09-18T10:00:20.001+10:00>  - In Pre-Hook, Arguments: {:a a, :b b}
    ;;> Instance: #<DateTime 2012-09-18T10:00:20.001+10:00>  - In Handle, Arguments: (:pre Pre-Hooked :a a :b b)
    ;;> Instance: #<DateTime 2012-09-18T10:00:20.001+10:00>  - In Post-Hook, Arguments:  {:result Result, :pre Pre-Hooked, :a a, :b b}

    (cj/stop!)

Thats all really... now go write your own handlers!

## The Details

Tasks can be added and removed on the fly through the `cronj` library interface and the library will then keep an eye out on the time. At the correct time that a task has been scheduled to start, the task handler will be launched in another thread. The actual polling loop is quite efficient and will only poll a maximum of twice every second with a 1ms timing error. Future improvements to the loop will hope to preempt the time that tasks should start and sleep until it is necessary to wake up.


### Overview:

Cronj is seperated into three basic concepts:

- Tasks are records that provide information about the task and keeps track of the running instances of the task.

- A Timesheet to strictly schedule and unschedule tasks according to a `tab` schedule as well as as provide functionality to easily manipulate groups of tasks.

- A Timekeeper to keep the time and trigger tasks at the time that it is scheduled to run.

<pre>
                                          timesheet
                  __...--+----+----+----+----+----+----+----+----+----+----+----+----+
         _..---'""      _|.--"|    |    |    |    |    |    |    |    |    |    |    |
        +-------------+'_+----+----+----+----+----+----+----+----+----+----+----+----+
        | task        |-     /                                              |
        |             |     /                     X                         |
        |    :id      |    /                    XXXXX              timesheet  methods
        |    :desc    |   /                    XXXXXXX          +-------------------------+
        |  ++:handler |  /                    XXXXXXXXX           contains-task? [id]
        | +++:tab       /                        XXX              select-task    [id]
        | || :enabled |/                         XXX              enable-task!   [id]
        +-++----------+                          XXX              disable-task!  [id]
          ||  ,-.                                XXX              trigger-task!  [id]
          |+-(   ) fn[dt & args]                 XXX              list-running-for-task  [id]
          |   `-'                                XXX              kill-all-running-for-task! [id]
         +-------------------------+             XXX              kill-running-for-task! [id tid]
         |  "* 8 /2 7-9 2,3 * *"   |             XXX
         +-------------------------+             XXX              list-all-tasks []
         |  :sec    [:*]           |             XXX              load-tasks!    [v]
         |  :min    [:# 8]         |             XXX              unschedule-all-tasks! []
         |  :hour   [:| 2]         |          XXXXXXXXX           schedule-task!   [id]
         |  :dayw   [:- 7 9]       |        XX         XX         unschedule-task! [id]
         |  :daym   [:# 2] [:# 3]  |      XX             XX
         |  :month  [:*]           |     X      cronj      X
         |  :year   [:*]           |    X                   X
         +-------------------------+    X     :thread       X
                                        X     :last-check   X        timekeeper methods
            cronj function               X    :interval    X    +-------------------------+
            --------------                XX             XX        running?     start!
           At every second.                 XX         XX          stopped?     stop!
           looks at task                      XXXXXXXXX            restart!
           list and triggers
           handler functions
           for each enabled
           task.
</pre>

### Tasks:

A "task" has the following attributes:

    - "id" and "desc" for meta description of the task
    - "handler", the actual procedure that provides the functionality for a task
    - arguments to the handler can be passed via the optional "args" attribute

A task does not have a concept of when it will run. It only responds when it is asked to run and will keep track of all the instances of the task that are running. It is defined as follows:

    (require '[cronj.data.task :as ct])
    (require '[clj-time.core :as t])
    (def task-10s (ct/task :10 "10s"
                         (fn [_] (println "I Last for 10 seconds")
                                 (Thread/sleep 10000))))

`task-10s` references a map:

    (println job-10s)
    ;; => {:enabled #[Atom@5148bd9e: true], :running [DynaRec], :args {}, :last-exec #[Atom@7e98f9c2: nil], :last-successful #[Atom@6d35707c: nil], :id :10, :desc 10s, :handler #user$fn}

It can be triggered using:

    (ct/exec! task-10s *id*)

where *id* has to be a unique identifier. In this case, and also in the case for cronj, we will use the the datetime. `running` lists the instances of the task that are still running:

    (ct/exec! task-10s (t/now))
    ;; => I Last for 10 seconds
    (ct/running task-10s) ;; within ten seconds of the exec! call
    ;; => (#<DateTime 2012-09-17T05:19:36.223Z>)


### Long Running Tasks

A task may be scheduled to run every 30 seconds, but may take up to a minute to finish. In this case, the multiple threads will be spawned and will be tracked. Below is an example:

    (ct/exec! task-10s :id-1)
    (ct/exec! task-10s :id-2)
    (ct/exec! task-10s :id-3)
    (ct/exec! task-10s :id-4)
    (ct/running task-10s) ;; within ten seconds of the three
    ;; => (:id-1 :id-2 :id-3 :id-4)

    ;; and then they can be killed individually or together:
    (ct/kill! task-10s :id-4)
    (ct/running task-10s)
    ;; => (:id-1 :id-2 :id-3)

    (ct/kill-all! task-10s)
    (ct/running task-10s)
    ;; => ()

The complete set of attributes for the task map are described:

    :id                The unique identifier for the task
    :desc              A description for the task
    :handler           The function that is called using `exec!`
    :pre-hook          Additional processing on the arguments prior to the execution of the handle
    :post-hook         Post result handling after the handle is run
    :enabled           If false, the task will not run
    :args              Additional arguments to the handler
    :running           All the instances of the task that are still running
    :last-exec         The id of the last instance called using `exec!`
    :last-successful   The id of the last instance that finished normally.

### Cronj

Enough about tasks, how do we go about using them as part of cronj? Tasks can be added to cronj in four ways, through a map or through a new task in combination with tab added in the task or later on:

    (ns examples.schedules
      (:require [cronj.data.task :as ct]
                [cronj.core :as cj] :reload))

    ;; Tab added first
    (cj/schedule-task!
        {:id 1
         :desc 1
         :handler #(println "Task 1: " %)
         :tab "0-60/4 * * * * * *"})

    (cj/schedule-task!
        (cronj.task/new 2 "2" #(println "Task 2: " %) :tab "1-60/4 * * * * * *"))

    ;;Tab added later
    (cj/schedule-task!
        {:id 3
         :desc 3
         :handler #(println "Task 3: " %)}
        "2-60/4 * * * * * *")

    (cj/schedule-task!
        (cronj.task/new 4 "4" #(println "Task 4: " %))
        "3-60/4 * * * * * *")

    (cj/start!)

    ;; Outputs:
    ;; Task 2:  #<DateTime 2012-09-19T10:49:37.000+10:00>
    ;; Task 3:  #<DateTime 2012-09-19T10:49:38.000+10:00>
    ;; Task 4:  #<DateTime 2012-09-19T10:49:39.000+10:00>
    ;; Task 1:  #<DateTime 2012-09-19T10:49:40.000+10:00>
    ;; Task 2:  #<DateTime 2012-09-19T10:49:41.000+10:00>
    ;; Task 3:  #<DateTime 2012-09-19T10:49:42.000+10:00>
    ;; Task 4:  #<DateTime 2012-09-19T10:49:43.001+10:00>

    (cj/stop!)


Controls for each task can be accessed through cronj.core:

    > (cj/unschedule-all-tasks!)
    > (cj/schedule-task!
       {:id :30s-task
        :desc "This is a 30 second task"
        :handler (fn [_] (Thread/sleep 30000))
        :tab "/5 * * * * * *"})

     > (cj/start!)

     > (cj/running?)
     ;;=> true

     > (pprint (cj/list-running-for-task :30s-task)) ;; after 10 secs
     ;;=>  (#<DateTime 2012-09-19T11:09:40.000+10:00>
     ;;     #<DateTime 2012-09-19T11:09:45.001+10:00>)

     > (pprint (cj/list-running-for-task :30s-task)) ;; after 20 secs
    ;;=>   (#<DateTime 2012-09-19T11:09:40.000+10:00>
            #<DateTime 2012-09-19T11:09:45.001+10:00>
            #<DateTime 2012-09-19T11:09:50.000+10:00>
            #<DateTime 2012-09-19T11:09:55.001+10:00>)

    > (pprint [(cj/last-exec-for-task :30s-task)  ;; should be 30s apart
               (cj/last-successful-for-task :30s-task)])
    ;;=>   [#<DateTime 2012-09-19T11:17:35.000+10:00>
            #<DateTime 2012-09-19T11:17:05.000+10:00>]

    > (cj/kill-all-running-for-task! :30s-task)
    ;;=>    #<InterruptedException java.lang.InterruptedException: sleep interrupted>
            #<InterruptedException java.lang.InterruptedException: sleep interrupted>
            #<InterruptedException java.lang.InterruptedException: sleep interrupted>
            #<InterruptedException java.lang.InterruptedException: sleep interrupted>
            #<InterruptedException java.lang.InterruptedException: sleep interrupted>
            #<InterruptedException java.lang.InterruptedException: sleep interrupted>

### Methods
The functions in `cronj.core` should be pretty self evident:

Basic:

    list-all-tasks []
    list-all-task-ids []
    load-tasks!    [v]
    unschedule-all-tasks! []
    schedule-task!   [task]
    schedule-task!   [task tab]
    reschedule-task!   [id tab]
    unschedule-task! [id]

Controls:

    stopped? []
    running? []
    start!   []
    stop!    []    stops the loop but lets all running tasks complete
    restart! []

    shutdown!! []  stops the loop, interruping and killing all unfinished task threads

Task Related:

    contains-task? [id]
    select-task    [id]
    enable-task!   [id]
    disable-task!  [id]
    trigger-task!  [id]
    list-running-for-task  [id]
    last-exec-for-task [id]
    last-successful-for-task [id]
    kill-all-running-for-task! [id]
    kill-running-for-task! [id tid]


## TODO:

Finish Tests
- Tab (Done)
- Task (More use cases)
- Timesheet (More use cases)
- Timekeeper
- Core (More use cases)
