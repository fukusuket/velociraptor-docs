---
title: Server.Monitor.Health
hidden: true
tags: [Server Event Artifact]
---

This is the main server health dashboard. It is shown on the
homescreen and enabled by default on all new installs.

You may edit this artifact to customize your server dashboard.

Alternatively, edit the Welcome screen at the
`Server.Internal.Welcome` artifact.


<pre><code class="language-yaml">
name: Server.Monitor.Health
description: |
  This is the main server health dashboard. It is shown on the
  homescreen and enabled by default on all new installs.

  You may edit this artifact to customize your server dashboard.

  Alternatively, edit the Welcome screen at the
  `Server.Internal.Welcome` artifact.

type: SERVER_EVENT

sources:
  - name: Prometheus
    query: SELECT sleep(time=10000000) FROM scope()

reports:
  - type: SERVER_EVENT
    # Only allow the report to run for 10 seconds - this is plenty for
    # the GUI.
    timeout: 10
    parameters:
      - name: Sample
        default: "6"

    template: |
      {{ define "CPU" }}
        LET SampledData &lt;= SELECT * FROM sample(
             n=atoi(string=Sample),
             query={
              SELECT _ts as Timestamp,
                  CPUPercent,
                  int(int=MemoryUse / 1048576) AS MemoryUse_Mb,
                  TotalFrontends
              FROM source(source="Prometheus",
                  start_time=StartTime, end_time=EndTime,
                  artifact="Server.Monitor.Health")
        })

        LET Stats &lt;= SELECT count() AS Count,
            timestamp(epoch=min(item=Timestamp)) AS MinTime,
            timestamp(epoch=max(item=Timestamp)) AS MaxTime,
            timestamp(epoch=StartTime) AS StartTime
        FROM SampledData
        GROUP BY 1

        // Include a log for verification. Last data should always be
        // very recent and sample should be passed properly.
        LET _ &lt;= log(message="Graphs cover times from %v (%v). Actual data available from %v (%v) to %v (%v) with %v rows. Data is sampled every %v samples.", args=[
           Stats[0].StartTime.String, humanize(time=Stats[0].StartTime),
           Stats[0].MinTime.String, humanize(time=Stats[0].MinTime),
           Stats[0].MaxTime.String, humanize(time=Stats[0].MaxTime),
           Stats[0].Count, Sample])

        SELECT * FROM SampledData
      {{ end }}

      {{ define "CurrentConnections" }}
        SELECT * FROM sample(
             n=atoi(string=Sample),
             query={
               SELECT _ts as Timestamp,
                  client_comms_current_connections
               FROM source(source="Prometheus",
                           start_time=StartTime, end_time=EndTime,
                           artifact="Server.Monitor.Health")
        })
      {{ end }}

      {{ $time_rows := Query "SELECT timestamp(epoch=now()) AS Now FROM scope()" | Expand }}
      ## Server status @ {{ Render ( Get $time_rows "0.Now" ) }}

      &lt;p&gt;The following are total across all frontends.&lt;/p&gt;
          &lt;span class="container"&gt;
            &lt;span class="row"&gt;
              &lt;span class="col-sm panel"&gt;
               CPU and Memory Utilization
               {{- Query "CPU" | TimeChart "RSS.yaxis" 2 -}}
              &lt;/span&gt;
              &lt;span class="col-sm panel"&gt;
               Currently Connected Clients
               {{- Query "CurrentConnections" | TimeChart "RSS.yaxis" 2 -}}
              &lt;/span&gt;
            &lt;/span&gt;
          &lt;/span&gt;

      ## Current Orgs
      {{ define "OrgsTable" }}
         LET ColumnTypes &lt;= dict(ClientConfig='url')
         LET OrgsTable = SELECT Name, OrgId,
                         upload(accessor='data', file=_client_config,
                                name='client.'+OrgId+'.config.yaml') AS _Upload
         FROM orgs()

         SELECT Name, OrgId, link_to(upload=_Upload) AS ClientConfig
         FROM OrgsTable
      {{ end }}

      {{ Query "OrgsTable" | Table }}

      ## Disk Space

      {{ Query "SELECT * FROM Artifact.Generic.Client.DiskSpace()" | Table }}

      ## Users

      {{ define "UserPermissions" }}
        SELECT name, effective_policy AS _EffectivePolicy,
               join(array=roles, sep=", ") AS Roles
        FROM gui_users()
      {{ end }}

      {{ Query "UserPermissions" | Table }}

      ## Server version

      {{ Query "SELECT server_version FROM config" | Table }}

</code></pre>

