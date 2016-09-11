---
layout: post
title:  "Working with Calendar in the Google API Gem"
date:   2016-09-10 5:35:09 -0500
display-category: "Ruby on Rails"
type: "blog"
description: "Working with the refresh token and scheduling events."
excerpt_separator: <!--more-->
tags: [ruby-on-rails, api, little-clerk]
---
I have started building a scheduling web app in my free time. Eventually a user will be able to choose which calendar you will want to schedule to but I am starting with Google Calendar. Because the gem is in alpha the documentation for the Google API and Calendar are sparse. So I wanted to create a short synopsis of my process getting it working.
<!--more-->

The app's name is Little Clerk. The general flow is I will receive a forwarded email, parse the event out of it and schedule it to the user's Google Calendar. This seemed like a good next-step to get myself to build an app that interacts with an API and doesn't just need to do CRUD operations. I started with getting my interation with Google's API down. I decided to use their ruby [gem][api-gem] to handle the raw interactions with the API. NOTE: This gem is in alpha so it can (and probably will) change. 

This post won't cover the entirety of the process of setting up the gem to work on your app. I was fortunate to find the following [guide][how-to-post] that helped me set it up.

###### For this post I will be focusing on the following pieces:

1. Implementing using the refresh token so you do not have to have the user re-authenticate every hour.
2. Scheduling an actual event to the user's calendar


## Implementing the refresh token

If you implement the gem in the way the [guide][how-to-post] tells you to your access to the user's calendar will only last for a certain amount of time and at that point the token will expire and the user would have to reauthenticate. Luckily in the return data from the initial authorization there is also a refresh token. If you set your application to request offline access this refresh token will allow us to get a new token when necessary. 

When I find the user in `User#find_for_google_oauth2` I storing some additional information:

- token_refresh_date: The datetime the token provided will expire. I check this to see if I need to request a new token.
- refresh_token: This is the actual token you will use to request a new token

{% highlight ruby %}
  def self.find_for_google_oauth2(access_token, signed_in_resource=nil)
    data = access_token.info
    user = User.find_by(email: data.email)
    if user
      user.provider = access_token.provider
      user.uid = access_token.uid
      user.token = access_token.credentials.token
      user.token_refresh_date = Time.zone.at(access_token.credentials.expires_at).to_datetime
      if access_token.credentials.refresh_token
        user.refresh_token = access_token.credentials.refresh_token
      end
      user.save
      user
    else
      redirect_to new_user_registration_path, notice: "Failed to find for Google Auth."
    end
  end
{% endhighlight %}

The `expires_at` is supplied in Unix time so I convert it to a datetime and store it in the user. I am checking if `refresh_token` is available just in case the user had already authenticated. Once you have the refresh token and when it expires we can work on the authorization.

I stored the handling of the authorization in a separate class from the class in charge of scheduling:

{% highlight ruby %}
TOKEN_CREDENTIAL_URI = "https://accounts.google.com/o/oauth2/token"
REDIRECT_URI = 'http://localhost:1234/users/auth/google_oauth2/callback'
AUTHORIZATION_URI = "https://accounts.google.com/o/oauth2/auth"

class GoogleAuthorizer

  def initialize(user, client: Signet::OAuth2::Client.new)
    @client = client
    @user = user
    @client.access_token = user.token
    @client.client_id = ENV['GOOGLE_CLIENT_ID']
    @client.client_secret = ENV['GOOGLE_CLIENT_SECRET']
    @client.token_credential_uri = TOKEN_CREDENTIAL_URI
    @client.redirect_uri = REDIRECT_URI
    @client.authorization_uri = AUTHORIZATION_URI
    @client.grant_type = "refresh_token"
    @client.refresh_token = user.refresh_token
    @client.scope = "profile, calendar, offline"

    if refresh_needed?
      refresh_access_token
    end
  end

  def authorization
    @client
  end

  private

  def refresh_access_token
    token = authorization.fetch_access_token!
    @user.token = token["access_token"]
    @user.token_refresh_date = Time.zone.now + token["expires_in"]
    @user.save
  end

  def refresh_needed?
    if @user.token_refresh_date
      @user.token_refresh_date < Time.zone.now
    else
      true
    end
  end
end
{% endhighlight %}

A couple things are happening here:

- I am setting all the usual necessary information into Signet.
- In addition to the normal information I am setting the grant_type to refresh_token and setting the refresh_token to the stored refresh token.
- Once the object is set up I check if a refresh is necessary by comparing the refresh date to now.
- If a refresh is needed I call `fetch_access_token!` and set the new token and token_refresh_date for the user

Once the authorization is set up we can move on to scheduling an event.

## Scheduling an event

The event scheduling is relatively easier. There are only a couple things we have to do.

{% highlight ruby %}
require 'google/apis/calendar_v3'

class GoogleEventCreator

  attr_accessor :service, :client

  def initialize(client:)
    @client = client
    @service = Google::Apis::CalendarV3::CalendarService.new
    @service.authorization = @client.authorization
  end

  def create_event(start_date_time:, end_date_time:, summary:, location:)
      event = calendar::Event.new(summary: summary,
                              location: location,
                                start: parse_time(start_date_time),
                                end: parse_time(end_date_time))
      event = @service.insert_event('primary', event, send_notifications: true)
  end

  private

  def calendar
    @_calendar = Google::Apis::CalendarV3
  end

  def parse_time(time, time_zone= "US/Central")
    time = Time.parse(time).utc.iso8601
    calendar::EventDateTime.new(date_time: time, time_zone: time_zone)
  end
end

{% endhighlight %}

First we will initialize an instance of the Calendar service. Then pass the Signet library to the new object.

When we create the event we have to supply the event information. Create a new event from the API client and pass in the required information. The start and end datetimes have be formatted just so. Google [expects][datetime-info] them to be ISO 8601. So we will take the time and parse and convert it.

Once the Event has been created we can insert it. Currently I am hard-coding the 'primary' calendar for the user as well as whether to send_notifications. In the future this will be driven by the Rule that causes these events to be created. But that is for another blog post.


Here are the big To Dos that I will to get to when I have time:

- Transition to [Google Auth][google-auth] from Signet. The API gem is planning on moving towards that as the authorization library.
- I need to set up logic to watch for the [following][issues]:
	- The user has revoked access.
	- The refresh token has not been used for six months.
	- The user changed passwords and the token contains Gmail, Calendar, Contacts, or Hangouts scopes.
	- The user account has exceeded a certain number of refresh token requests.

Until then. Stay tuned, I will be making a lot more updates to this applicaiton in the coming months and I am planning on documenting it here. 

[api-gem]:      https://github.com/google/google-api-ruby-client
[documentation]:   https://
[how-to-post]: http://readysteadycode.com/howto-access-the-google-calendar-api-with-ruby
[datetime-info]: https://developers.google.com/schemas/formats/datetime-formatting
[google-auth]: https://github.com/google/google-auth-library-ruby
[issues]: https://developers.google.com/identity/protocols/OAuth2#expiration

