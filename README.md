## A Guide to Uploading Images to AWS (SDK 2) with Rails 4.2.4, Paperclip 4.3, and ImageMagick on localhost applications
<br/>

It was difficult to find a single resource that had correct and up-to-date information on how to make all of this work, so I've created this guide to show what worked for me. If you have any questions/comments feel free to reach me <a href='https://twitter.com/dgempler' target='\_blank'>on Twitter</a> or make a pull request with any proposed changes.

 I found that using AWS's S3, Paperclip 4.3, and ImageMagick worked best for my localhost application. There are changes that need to be made if you plan on putting your app up on Heroku (or adding a front-end framework like Angular), which I plan on addressing at some point in the future.

### Getting Started with Amazon Web Services

 - [Create an account](https://aws.amazon.com)
 - Choose their [Simple Storage Service (S3)](aws.amazon.com/s3) option (this is currently free for 12 mo. after signup, with [limits](https://aws.amazon.com/free/))
 - Choose S3 from the online AWS console, and create a bucket. Choose a region. More on regions and their endpoints [here](http://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region).  
 - Then create a user in (IAM) Security Credentials. Get there by clicking on your name in the nav bar, then Security Credentials in the dropdown, then select Users from the dashboard on the left side.
  - Attach a Policy to that user - __Administrator Access__ (Permissions tab)
 - Create your access key, save it with your id in environment (a gitignore-d .env file, or adding to your .zshrc / zshenv file, etc):
  `ENV['AWS_ACCESS_KEY_ID']` and `ENV['AWS_SECRET_ACCESS_KEY']`
 - While you're at it, throw in three more env. variables that you'll need later: `ENV[S3_BUCKET]` with your bucket name, your chosen `ENV[AWS_REGION]` and `ENV[AWS_ENDPOINT]`. See regions and endpoints link above.
  - Actual syntax in env. file will be `export AWS_REGION=us-west-2` or `export AWS_ENDPOINT=s3-us-west-2.amazonaws.com`

### Setting up Paperclip and ImageMagick

**[Paperclip](https://github.com/thoughtbot/paperclip)** is an easy file attachment library for ActiveRecord. It helps get your uploaded images processed (via ImageMagick) and sent on to AWS.

`gem "paperclip", "~> 4.3"`


_Note: Assuming you will be using `gem 'aws-sdk', '~> 2'` (as specified below), you must use the master branch of Paperclip, as paperclip 4.3 does not support aws-sdk version 2. Link it in your gemfile as follows:_

`gem "paperclip", git: "git://github.com/thoughtbot/paperclip.git"`

**[ImageMagick](http://git.imagemagick.org/repos/ImageMagick)** will process your images (resize, change formats, etc.) before Paperclip sends them to Amazon. This means that they will automatically get saved in a temporary location in your project directory, so it won't work well with large file uploads.

``
brew install imagemagick
``
(or [install from source](http://www.imagemagick.org/script/install-source.php))

*Note: Different settings will be required for Heroku deployment. There is also the option of sending images directly to AWS without processing, i.e., everything done on the front end, but the processing you can do in the background is limited. If using Angular, make the user do it with [ng-file-upload](https://github.com/danialfarid/ng-file-upload).*

Go through the [Quick Start](https://github.com/thoughtbot/paperclip#quick-start) guide on Paperclip. Set up your migration for your image column separately, i.e., add the image/avatar/whatever column to your model in a separate migration, as shown [here](https://github.com/thoughtbot/paperclip#migrations) in the Quick Start guide. They can still be migrated together in a single `db:migrate` along with the rest of your migrations.

### Configuration, Security, and Syncing Paperclip with AWS

Add the AWS gem:

`
gem 'aws-sdk', '~> 2'
`

Don't forget to `bundle install` and restart your rails server after modifying the Gemfile!

Create `aws.rb` and `paperclip.rb` config files in `config/initializers`

``` ruby

# /config/initializers/aws.rb
Aws.config.update({
  region: ENV['AWS_REGION'],
  credentials: Aws::Credentials.new(ENV['AWS_ACCESS_KEY_ID'], ENV['AWS_SECRET_ACCESS_KEY']),
})
```

``` ruby
# /config/initializers/paperclip.rb
Paperclip::Attachment.default_options[:s3_host_name] = ENV['AWS_ENDPOINT']
```

Then, set paperclip defaults and options in `config/environments/development.rb`. You can alternatively add the defaults to `config/application.rb`, but the last line (options) may not work if it's not in `development.rb`

``` ruby
config.paperclip_defaults = {
  :storage => :s3,
  :s3_region => ENV['AWS_REGION'],
  :s3_credentials => {
    :bucket => ENV['S3_BUCKET'],
    :access_key_id => ENV['AWS_ACCESS_KEY_ID'],
    :secret_access_key => ENV['AWS_SECRET_ACCESS_KEY']
  }
}

Paperclip.options[:command_path] = "/usr/local/bin"
```

The path on the last line comes from ImageMagick. If you haven't done so already, type `which convert` anywhere in your terminal. If it was installed correctly, it will return a path. Drop the `/convert` off the end of it and plug it in to `Paperclip.options[:command_path]`.

### Upload the images

In your forms, make sure to include `multipart: true`:

``` erb
<%= form_for @human, html: {multipart: true} do |f| %>
```

And if the image you are uploading isn't associated with a model:

``` erb
<%= form_tag({:action => :upload}, :multipart => true) do %>
  <%= file_field_tag 'image' %>
<% end %>
```

Reference the [Paperclip README](https://github.com/thoughtbot/paperclip#edit-and-new-views) for more details on forms.

An additional `'file'` parameter will be sent along with your other specified params.

In your controller, don't permit any extra parameters. Paperclip magic will take care of it.

``` ruby
def human_params
  params.require(:human).permit(:name, :image)
end
```

In your server logs, you might see this after the incoming parameters are logged: `Unpermitted parameters: file`.

### And that's it!

You should now be able to see uploaded images in your bucket from the AWS console.

You can access your images in rails/erb by using the [url property](https://github.com/thoughtbot/paperclip#show-view) created by Paperclip.

### Issues you may run into

 - ##### Don't forget to:

 `bundle` <br/>
 restart `rails s` <br/>
 `rake db:create; rake db:migrate`



 - **Paperclip::Errors::NotIdentifiedByImageMagickError**

You likely have problems saving your images due to a bad install of your post-processor, ImageMagick.
Try removing styles from your model (if you specified any) and see if it works:
``` ruby
has_attached_file :image, styles: {
  thumb: '100x100>',
  medium: '300x300>'
}
```
to
``` ruby
has_attached_file :image
```
I was able to solve this problem by uninstalling and reinstalling ImageMagick. You might have to do it a few times. If you used brew to install it:


`brew uninstall imagemagick`

`brew install imagemagick`



### Other Resources - some helpful, some not

[This Heroku guide](https://devcenter.heroku.com/articles/paperclip-s3) was the most helpful resource I found for getting started (if planning to put up on Heroku), but was still missing some necessary information.

[AWS docs for Ruby](http://docs.aws.amazon.com/sdkforruby/api/index.html)
