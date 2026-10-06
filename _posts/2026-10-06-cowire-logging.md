---
layout: post
title: "Collecting Cowrie Logs"
description: >
  Completing the link between Cowrie and Graylog. Our first pipeline
image: /assets/img/graylog/graylog-logo.jfif
---

Now that the Cowrie server is up and running, how can we connect this to Graylog?
{:.lead}

Cowrie actually has a built-in way to send logs over to our Graylog server. This is where the magic of SIEM tools really starts to shine.

Essentially, you need to create a data stream to get data from one point to another in your environment. Log buses (such as Vector, Kinesis, or Kafka) are usually the industry standard for this kind of thing, since they can bundle a whole host of logs from a single server and send them over in a single data stream, but that is a post for another time.

What we are going to do is set up a single data stream with Cowrie's built-in output and send the logs over to Graylog directly for ingestion.

Here's the plan. Graylog needs a few pieces in place before Cowrie's logs show up somewhere useful: an **input** to receive them, an **index set** to store them, a **stream** to sort them, a **routing rule** to fill that stream, and a **pipeline** to turn raw JSON into searchable fields. The order matters, so we'll build them one at a time and test as we go.

* toc
{:toc .large-only}

## Part 1: Create the GELF Input in Graylog

First, Graylog needs something listening for Cowrie's logs. Graylog uses **inputs** for this, and the format we'll use is GELF (Graylog Extended Log Format), which Cowrie can speak natively.

Go to **System → Inputs**, select **GELF HTTP** from the dropdown, and click **Launch new input**. Use these settings:

- **Title:** `Cowrie GELF`
- **Bind address:** `0.0.0.0`
- **Port:** `12201`

The bind address is important. `0.0.0.0` tells Graylog to listen on all interfaces so Cowrie can reach it from another machine. If you leave it at `127.0.0.1`, Graylog will only accept traffic from itself, and your honeypot logs will never arrive.

Launch it and confirm the input shows a green **RUNNING** state.

**Watch for port collisions.** If you already have another input bound to port 12201 (a GELF UDP input, for example), the two will conflict, and messages will silently fail to arrive. Keep exactly one input on that port, and make sure its protocol matches Cowrie's, which is HTTP. There will be an error if you try to start an input on a port that is already in use.
{:.note}

## Part 2: Point Cowrie at Graylog

With Graylog listening, we can tell Cowrie where to send its logs. On the honeypot, open Cowrie's config file (`etc/cowrie.cfg` inside your Cowrie directory) and find or add the Graylog output section:

```ini
[output_graylog]
enabled = true
url = http://<graylog-ip>:12201/gelf
```

Replace `<graylog-ip>` with your Graylog server's address. The port matches the input we just created, and `/gelf` is the path Graylog's GELF HTTP input listens on.

Save the file and restart Cowrie so it picks up the change:

```bash
bin/cowrie restart
```

If the honeypot and Graylog sit on different machines, make sure port 12201 is allowed through any firewall between them.
{:.note}

### Checkpoint: Are Logs Arriving?

Before building anything else, SSH into the honeypot from another machine and throw in a junk username and password. Then go back to **System → Inputs** in Graylog and check that the `Cowrie GELF` input shows some throughput.

For now, those messages land in Graylog's **Default Stream**, mixed in with everything else. That's expected. The next few parts give them a proper home.

## Part 3: Create a Dedicated Index Set

Go to **System → Indices → Create index set**. Give it the title `Cowrie` and the index prefix `cowrie` (lowercase, no spaces). The default rotation and retention settings are fine for a lab.

Why create a separate index set? Different logs you ingest may have different retention requirements, ingest resource requirements, etc. It is always good practice to have a different index set for each group of logs that you're ingesting.

## Part 4: Create the Stream and Route Messages Into It

A **stream** is how Graylog sorts incoming messages into categories. We'll create one for the honeypot and write a rule that sends Cowrie's messages into it.

### Create the Stream

Go to **Streams → Create Stream**. Title it `Cowrie Honeypot`, and set its **Index Set** to the `Cowrie` index set from Part 3. This is the link that routes anything in the stream into that index.

### Add the Routing Rule

Open the stream and click **Manage Rules**. Here's the part that trips people up: the rule has to match on a field that exists on the raw message *before* any parsing happens.

At this point, Cowrie's data is still one big JSON blob sitting inside the `message` field. Fields like `eventid` or `username` don't exist yet, so a rule built on them will never match. Instead, route on the input itself. Graylog has a stream-rule condition for matching the input a message came from, so select your `Cowrie GELF` input there.

Set the stream to require that **at least one rule** matches, and keep just this one rule.

### Start the Stream

New streams are paused by default. Click **Start Stream**, or nothing will be processed.

### Test the Routing

Routing is not retroactive. It only applies to messages that arrive *after* the rule is live, so the test messages from Part 2 will stay in the Default Stream. Send a fresh SSH attempt to the honeypot and check that it shows up in the `Cowrie Honeypot` stream.

If the input shows throughput but the stream stays empty, open one of the new messages. If the detail view says it was **routed into Default Stream**, your stream rule isn't matching. Go back and make sure you're routing on the input.

### Remove Matches from the Default Stream (Optional)

Once routing works, enable **Remove matches from Default Stream** in the stream settings. This keeps Cowrie logs living only in their own stream and index instead of being duplicated into the Default Stream.

Do this *after* you've verified routing, so you're only changing one thing at a time. If something breaks, you'll know exactly what caused it.

## Part 5: Our First Pipeline: Parsing Cowrie's JSON

Logs are flowing into the right place, but if you open one up, you'll notice something annoying: all the good stuff is crammed into a single JSON string inside the `message` field. `eventid`, `username`, `password`, and `src_ip` aren't separate, searchable fields yet, which makes the data pretty much useless for searching or dashboards.

This is where our first pipeline comes in. A pipeline rule will parse that JSON and promote every key to its own top-level field.

### Create the Rule

Go to **System → Pipelines → Manage Rules → Create Rule** and paste in the following:

```
rule "Parse Cowrie JSON"
when
  has_field("message")
then
  let json_tree = parse_json(to_string($message.message));
  let json_fields = to_map(json_tree);
  set_fields(json_fields);
end
```

Here's what's happening. The `when` block fires for any message that has a `message` field. In the `then` block, `parse_json` reads the message text as JSON, `to_map` converts the result into a key-value map, and `set_fields` takes every key in that map and turns it into its own field on the message.

### Create the Pipeline

Go to **System → Pipelines → Manage Pipelines → Add new pipeline** and title it `Cowrie Processing`. Edit **Stage 0** and add the `Parse Cowrie JSON` rule to it.

### Connect the Pipeline to the Cowrie Stream

On the pipeline page, click **Edit connections** and connect it to the `Cowrie Honeypot` stream (not the Default Stream). This scopes the parsing to honeypot data only.

### Check the Processor Order

Go to **System → Configurations → Message Processors**. Make sure both the **Message Filter Chain** (which handles stream routing) and the **Pipeline Processor** are enabled, and that the pipeline runs after message extraction. If the order is wrong, the pipeline may run before the message has been routed to the Cowrie stream and never touch it.

### Test the Pipeline

Just like routing, pipeline rules only apply to messages that arrive after the rule is live. Send one more fresh SSH attempt to the honeypot, then expand the new event in Graylog. You should now see `eventid`, `username`, `password`, and `src_ip` listed as their own fields.

**A note for production:** `has_field("message")` matches basically every message Graylog receives. Non-JSON messages will just fail the parse harmlessly, which is fine for a single-purpose honeypot. In a busier environment, though, you'd want to scope the rule to the Cowrie input or stream so you're not wasting processing on every log that passes through.
{:.note}

## Wrapping Up

We now have a complete path from Cowrie to Graylog. Every login attempt is shipped over GELF, lands in its own stream and index, and gets broken out into searchable fields by our first pipeline. That sets us up nicely for the fun part: building searches and dashboards to see what attackers are actually doing.
