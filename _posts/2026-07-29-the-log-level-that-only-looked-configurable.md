---
layout: post
title: "The log level that only looked configurable"
date: 2026-07-29
author: Miguel Santos
tags: [airflow]
pr_status: merged
---

The Stackable Airflow operator generates Airflow's logging config from your CRD. You set a level, it renders the Python. An open issue asked for one specific thing: make the log level of the `task` handler configurable. There was even a comment in the source saying it needed doing.

I started by trying to prove the comment was wrong. It looked wrong. The CRD has a `loggers` map, the docs show an `airflow.task` entry in it, and setting that entry does change what you see in the Airflow UI. So the feature seemed to already exist and nobody had noticed.

## Handlers filter, loggers gate

It does work, but not as the same thing. Python's `logging` has two thresholds on the path from a log call to an output, and they are not interchangeable.

```python
logger = logging.getLogger('airflow.task')   # threshold 1: the logger
handler = logger.handlers[0]                 # threshold 2: the handler
```

A record is dropped at the logger before any handler is consulted. If it survives, each handler applies its own level independently. So the logger is a gate on the whole path, and a handler level filters one destination.

That distinction is the entire issue. Airflow's `task` handler is what feeds the log view in the web UI. The other handlers write to files, which is what the Vector agent ships off the node. If you raise the `airflow.task` logger to suppress noise in the UI, you have also stopped those records from ever reaching the files. The thing people actually wanted, from a second issue on the same topic, was DEBUG in the files and in the log aggregator while the UI stays readable at INFO. A logger level cannot express that. Only a handler level can.

So the comment in the source was right and my read of it was wrong. Good thing to establish before writing a line of code.

## Building the obvious thing

The CRD shape looked like the blocker. Levels come from a struct with named fields:

```rust
pub struct AutomaticContainerLogConfig {
    pub loggers: BTreeMap<String, LoggerConfig>,
    pub console: Option<AppenderConfig>,
    pub file: Option<AppenderConfig>,
}
```

`console` and `file` are fields. `loggers` is a map, so any key you invent works without changing the struct, which is exactly why the `airflow.task` workaround needed nothing upstream. But a `task` field next to `console` and `file` is a struct change, and that struct lives in a shared crate. I checked: thirteen operators consume it.

I built it anyway. Added the field, added it to the hand-written merge implementation, updated twenty-five struct literals the compiler pointed me at, regenerated the CRDs, wrote the tests, opened the upstream PR, and wired the operator side to it. It worked. I verified it end to end by rendering the generated Python and running it through `dictConfig`, because a test asserting on generated source does not prove the level takes effect at runtime.

## The reply that made it smaller

A maintainer answered roughly: I would rather not add a field to every product's CRD when only Airflow uses it. Then he offered an alternative, and noted that reading the docs he would have expected `airflow.task` in `loggers` to already be exactly that knob.

Which is the point I had walked straight past. If the level has to come from somewhere, and `loggers` already accepts any key, and the docs already tell people to put `airflow.task` there, then the value is already in the CRD. Nothing needs adding. What needed changing was what the operator does with it: send it to the handler instead of to the logger.

```rust
fn task_log_level(log_config: &AutomaticContainerLogConfig) -> LogLevel {
    log_config
        .loggers
        .get(TASK_LOGGER)
        .map(|logger| logger.level)
        .unwrap_or(LogLevel::INFO)
}

fn task_logger_level(log_config: &AutomaticContainerLogConfig) -> LogLevel {
    cmp::min(LogLevel::INFO, task_log_level(log_config))
}
```

The second function is the part worth explaining. The handler gets whatever you asked for. The logger gets the lower of INFO and your level, never the higher. Asking for DEBUG has to open the logger up too, or it would drop the records before the handler could pass them. Asking for ERROR must not raise the logger, because the console and file handlers still want everything from INFO up. Clamping in one direction is what lets one knob mean the right thing in both.

He also offered a second option: just reuse the `file` level for the task handler, as a close-enough approximation. I pushed back on that one. It collapses the exact distinction the issue is about, and it would silently start showing DEBUG in the UI to anyone who has `file` set to DEBUG today. Agreeing with the reviewer on the good idea does not mean agreeing with all of them.

## What the rewrite cost

The CRD version touched two repositories, needed a release of the shared crate before it could merge, changed a struct thirteen operators depend on, and added 88 keys to the generated CRD schema. The rewrite touches three files in one repository and changes no schema at all. Same feature.

There is one honest wart, and I put it in the PR rather than in a footnote. This reinterprets a key that already does something. Today, raising `airflow.task` to ERROR suppresses records everywhere; afterwards it only quietens the UI. That is the improvement, and it is also a behaviour change, and someone relying on the old side effect to cut log volume would notice.

## The reviewer found the one thing I had not run

The rework got a CHANGES_REQUESTED, and it was not about the design. It was about the example I had written into the docs.

I had documented `file` at DEBUG with `airflow.task` at INFO, and claimed that wrote DEBUG to the log files and to Vector while the UI stayed at INFO. It does not do that. With `airflow.task` at INFO the logger sits at INFO, so the DEBUG records are discarded there, before the file handler ever sees them. The clamp I was so pleased with is exactly what makes that combination impossible.

The mechanism I had explained in the PR was right. The example I built on top of it was wrong, and the reason it survived is embarrassingly simple: I verified the mechanism by running the generated config through `dictConfig`, and I verified the example by reading it. So the fix was to pick an example that actually works, `airflow.task` at WARN, which quietens the UI while console and file keep receiving INFO, and to run that one too.

Below INFO the two are coupled and there is no way around it. A logger throws records away before any of its handlers can filter them, so lowering the UI on its own cannot work, no matter how it is implemented. At or above INFO they diverge and only the UI goes quiet. The docs now say both directions instead of implying they are symmetric.

The tests changed shape as well. Instead of one case per level scattered around, there is a table listing all seven levels next to the resulting handler level and logger level, so the asymmetry is visible in one place. I checked the table actually guards something by removing the clamp: five tests go red.

## The takeaway

I spent most of the effort on the version that got thrown away, and the reason is not that the design was wrong. It is that I accepted the framing in my own head, that a handler level needs a handler field, and never asked whether the value could come from somewhere that already existed. The maintainer did not know Airflow's logging internals better than I did by then. He just had not adopted my constraint.

Worth noticing too that he arrived at the answer by reading the documentation and saying what he expected to be true. When a reviewer tells you what they assumed the feature already did, that is not confusion to correct. It is a fairly direct hint about where the feature belongs. And the second lesson is narrower but cost me a round of review: run the exact example you put in the docs. A verified mechanism is not a verified example. The change is in [stackabletech/airflow-operator#829](https://github.com/stackabletech/airflow-operator/pull/829), now merged.
