# Armory Spinnaker Slack Integration

### This configuration will setup Spinnaker to notify slack when a deployment event has happened such as (pipeline start, pipeline complete, and when a Maunal Judgement is required to procced as an examples).

1. create slack channel to recieve messages

2. create slack app with OAuth Token for Spinnaker to authenticate to

3. add slack app to the newly created slack channel

4. configure Spinnaker through Halyard to apply config

Here are the halyard commands required to configure Spinnaker to forward messages via Slackbot associated with the Slack App and Channel.

```code 
hal config notification slack edit --bot-name [SLACK_BOT_NAME] --token [OATH_TOKEN_STARTING WITH "xobo-"]
```
```code
hal config notification slack enable
```
```code
hal deploy apply
```

Validate Slack notifications by configuring a manual judgement stage with notification config.


### Congratulations!

You have setup Spinnaker Slack integration.  Now teams can be notified on events for their deployment pipelines.
