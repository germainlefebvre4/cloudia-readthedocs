# GCP Carbon Footprint

## Principles

Retrieving Cloud Projects Carbon Footprint requires actions on specified Cloud Projects. Actions can be realized directly from the root account which has access to all the projects throug AWS Cloudformation StackSet.

## Configuration

### Create BigQuery DataTransfert

Here is the official documentation about configuring BigQuery DataTransfert for Carbon Footprint Export: [https://cloud.google.com/carbon-footprint/docs/export](https://cloud.google.com/carbon-footprint/docs/export).

Make sure to enable the enable the "Export carbon footprint data" feature on the Google Project where you want to collect the data. **This feature enabling is not enough highligthed in the documentation**.

### Backfill data for previous months

The Carbon Footprint data is not backfilled for previous months. You need to backfill the data manually.
