---
title: 'Controlling the Syncing of a Database ClickPipe'
description: 'Doc for controllling the sync a database ClickPipe'
slug: /integrations/clickpipes/mysql/sync_control
sidebar_label: 'Controlling syncs'
---

import edit_sync_button from '@site/static/images/integrations/data-ingestion/clickpipes/postgres/edit_sync_button.png'
import create_sync_settings from '@site/static/images/integrations/data-ingestion/clickpipes/postgres/create_sync_settings.png'
import edit_sync_settings from '@site/static/images/integrations/data-ingestion/clickpipes/postgres/sync_settings_edit.png'
import cdc_syncs from '@site/static/images/integrations/data-ingestion/clickpipes/postgres/cdc_syncs.png'
import Image from '@theme/IdealImage';

This document describes how to control the sync of a database ClickPipe (Postgres, MySQL etc.) when the ClickPipe is in **CDC (Running) mode**.

## Overview {#overview-mysql-sync}

Database ClickPipes have an architecture that consists of two parallel processes - pulling from the source database and pushing to the target database. The pulling process is controlled by a sync configuration that defines how often the data should be pulled and how much data should be pulled at a time. By "at a time", we mean one batch - since the ClickPipe pulls and pushes data in batches.

There are two main ways to control the sync of a database ClickPipe. The ClickPipe will start pushing when one of the below settings kicks in.

### Sync interval {#interval-mysql-sync}

The sync interval of the pipe is the amount of time (in seconds) for which the ClickPipe will pull records from the source database. The time to push what we have to ClickHouse is not included in this interval.

The default is **1 minute**.
Sync interval can be set to any positive integer value, but it is recommended to keep it above 10 seconds.

### Pull batch size {#batch-size-mysql-sync}

The pull batch size is the number of records that the ClickPipe will pull from the source database in one batch. Records mean inserts, updates and deletes done on the tables that are part of the pipe.

The default is **100,000** records.
A safe maximum is 10 million.

### An exception: Long-running transactions on source {#transactions-pg-sync}

When a transaction is run on the source database, the ClickPipe waits until it receives the COMMIT of the transaction before it moves forward. This with **overrides** both the sync interval and the pull batch size.

### Configuring sync settings {#configuring-mysql-sync}

You can set the sync interval and pull batch size when you create a ClickPipe or edit an existing one.
When creating a ClickPipe it will be seen in the second step of the creation wizard, as shown below:

<Image img={create_sync_settings} alt="Create sync settings" size="md"/>

When editing an existing ClickPipe, you can head over to the **Settings** tab of the pipe, pause the pipe and then click on **Configure** here:

<Image img={edit_sync_button} alt="Edit sync button" size="md"/>

This will open a flyout with the sync settings, where you can change the sync interval and pull batch size:

<Image img={edit_sync_settings} alt="Edit sync settings" size="md"/>

### Monitoring sync control behaviour {#monitoring-mysql-sync}

You can see how long each batch takes in the **CDC Syncs** table in the **Metrics** tab of the ClickPipe. Note that the duration here includes push time and also if there are no rows incoming, the ClickPipe waits and the wait time is also included in the duration.

<Image img={cdc_syncs} alt="CDC Syncs table" size="md"/>
