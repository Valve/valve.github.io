---
layout: post
title: "different SMTP settings for ActionMailer action"
date: 2013-07-03 16:48
comments: true
categories: [ruby,rails,actionmailer]
---

Sometimes you may want to set per-action SMTP settings that are different from site-wide settings.
You might try to set them in mailer:

<!--more-->

{% codeblock lang:ruby %}
class UserMailer < ActionMailer::Base
  ActionMailer::Base.smtp_settings = {address: 'some.domain'}

  def user_registered(user)
    mail(to: user.email, subject: 'You registered here!')
  end
end
{% endcodeblock %}

But this will override the smtp settings for the rest of mailers.

Instead, set them for the email instance:

{% codeblock lang:ruby %}
# assuming in controller action
mail = UserMailer.user_registered(current_user)
custom_smtp_settings = {address: 'some.domain', port: 25}
mail.delivery_method.settings.merge! custom_smtp_settings
# now this will send using custom SMTP settings
mail.deliver
{% endcodeblock %}

`rails-3.1`
