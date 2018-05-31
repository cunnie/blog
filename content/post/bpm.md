---
authors:
- cunnie
categories:
- BOSH
- bpm
date: 2018-05-28T16:16:22Z
draft: true
short: |
  Enhancing a BOSH release with BPM brings a number of advantages,
  not the least of which is BOSH job isolation. In this blog post we
  describe the steps we followed to include BPM in our nginx BOSH release.
title: How to BPM-ify a BOSH release
---

A [BOSH]() Director is a virtual machine (VM) orchestrator. A BOSH Director
deploys BOSH releases (specially-packaged applications) to VMs. A BOSH release
consists of zero or more jobs (services, e.g. nginx), which can be enhanced with
[BPM](). BPM has several features, but the one we are most interested in is
process isolation, which means that even if a job (service) is compromised, the
VM on which it runs is not necessary compromised.

"Job colocation" is a term which describes When BOSH jobs from two or more
releases are run on the same VM. For example, we often colocate our nginx web
server on our BOSH Director, but we have misgivings: if our webserver is
compromised, our entire BOSH Director is compromised. With BPM we can limit the
damage and not expose our Director

This blog post is directed towards BOSH release writers who would like to
incorporate BPM in their BOSH release.

In this blog post we describe the steps we go through to BPM-ify our BOSH nginx
release.

### 0. Modify the `monit` script

Modify the job's monit script. This is where we add `bpm` as a wrapper:

```diff
-  with pidfile /var/vcap/sys/run/nginx/nginx.pid
-  start program "/var/vcap/jobs/nginx/bin/ctl start"
-  stop program "/var/vcap/jobs/nginx/bin/ctl stop"
+  with pidfile /var/vcap/jobs/nginx/nginx.pid
+  start program "/var/vcap/jobs/bpm/bin/bpm start nginx"
+  stop program "/var/vcap/jobs/bpm/bin/bpm stop nginx"
```

Note: normally the PID file would be placed in
`/var/vcap/sys/run/bpm/nginx/nginx.pid`; however, we don't have such luxury, for
`nginx` writes the PID file, not BPM. This is because `nginx` adheres to an ancient
practice of UNIX daemons, to whit: they bind to the port with root privileges,
change process group, close most descriptors, de-escalate privileges, fork()
the "master" process, then exit.

### 1. Get rid of the `ctl` script

After converting to BPM, _The ctl script is completely ignored_;
delete it. It's useless.

```bash
git rm jobs/nginx/templates/ctl.sh
```

Remove references to the `ctl` script from the spec file. And, while we're
editing this file, include a reference to a new BPM template (`bpm.yml.erb`)
which we will cover in the next section:

```diff
 ---
 name: nginx
 templates:
-   ctl.sh: bin/ctl
+   bpm.yml.erb: config/bpm.yml
```

Note: BPM does not affect the lifecycle jobs (pre-start, post-start, and drain);
it only affects the monit start-and-stop programs (i.e. the `ctl` script).

### 2. Create Your BPM Configuration Template File

Create your BPM configuration file (`jobs/nginx/templates/bpm.yml.erb`). Ours
looks like this:

```yaml
---
processes:
  auctioneer:
    executable: /var/vcap/packages/nginx/sbin/nginx
    args:
      - -g
      - pidfile /var/vcap/data/nginx/nginx.pid
      - -c
      - /var/vcap/jobs/nginx/etc/nginx.conf
    limits:
      open_files: 100000
```

### 2. Include BPM In Your Sample Manifests

Be generous to your users; include BPM in your sample manifests.

```diff
 releases:
 - name: nginx
   version: latest
+- name: bpm
+  version: latest
```

### Troubleshooting

When the job isn't starting

https://cloudfoundry.slack.com/archives/C7A0K6NMU/p1527201139000348

> If you want to troubleshoot the process manually, you can also try `monit unmonitor ...` and then just use `bpm start ...` interactively to see what is happening.

### References

Canonical BOSH BPM documentation: https://bosh.io/docs/bpm/bpm/

Transitioning to BPM: https://github.com/cloudfoundry-incubator/bpm-release/blob/master/docs/transitioning.md

### Footnotes

When `bpm` isn't colocated, the job will fail with a cryptic:

```
Task 7 | 01:01:04 | Updating instance nginx: nginx/87b5e8a9-7ed4-44af-bc09-4a2141da8c4b (0) (canary) (00:01:21)
                   L Error: 'nginx/87b5e8a9-7ed4-44af-bc09-4a2141da8c4b (0)' is not running after update. Review logs for failed jobs: nginx
```
