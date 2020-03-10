# website
Hoot Works website

## Environment Setup

Clone the repository locally.
 
Working from the root of the project, install required software (assuming MacOS):

```
brew install hugo
```

## Development

`hugo --ignoreCache`

or

`hugo --disableFastRender --ignoreCache --watch server`


## Slack Configuration

#### Channels

Configure two channels:

* `#hootworks-contact` - Used to view messages submitted via the website contact form.
* `#hootworks-deployment` - Used to view success or failure of Netlify deployments.

#### Contact App

Configure a new `hootworks-contact` app which has a Webhook posting to the `#hootworks-contact` channel. 

This will be used by a Slack Integration form notification configured in Netlify.

#### Deployment App

Configure a new `hootworks-deployment` app which has a Webhook posting to the `#hootworks-deployment` channel. 

This will be used by Slack Integration deploy notifications in Netlify.

#### Asset Optimization

Configure the following optimizations:

* URLs: *Pretty URLs*
* CSS: *Bundle & Minify*
* JS: *Bundle & Minify*
* Images: *Lossless compression*

#### Deploy Notifications

Configure the following Slack Integrations:

* On deployment failure post to the `hootworks-deployment` app webhook configured in Slack.
* On deployment success post to the `hootworks-deployment` app webhook configured in Slack.

#### Domain Management

Configuration for Domains and HTTPS is based on standard Netlify instructions.

#### Form Notifications

Configure the following Slack Integrations:

* On `New form submission` to the `contact` form, post to the `hootworks-contact` app webhook configured in Slack.
