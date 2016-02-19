---
layout: post
title: Amazon IAM Policy for Uploading Files to S3 with Rails on Heroku
disqus: true
---

I recently updated a Rails project I was working on to use S3 for user-content hosting.
There are multiple places in my application where users can upload images,
and I wanted to start hosting those images on S3 rather than on the Heroku server itself.

I found lots of guides online for how to make this happen. It seemed like it was going
to be a very straight forward process. For the most part, it was, except for one obstacle
that took me a while to figure out. [Heroku provides their own guide](https://devcenter.heroku.com/articles/paperclip-s3)
for how to set this up. I started by following their instructions.

The next step is to configure my AWS account. I created a bucket in S3 called `my-bucket`,
and created an IAM user also called `my-bucket`. AWS provided me with an `access_key_id`
and a `secret_access_key` for this user that I will use to connect to S3 from my Rails application.

The final step, and the one that tripped me up, is to implement an IAM Policy in my AWS account
that grants the user `my-bucket` the appropriate permissions to upload files to `my-bucket`
in my S3 account. After reading lots of blogs and following multiple tutorials from Amazon,
I settled on this IAM policy:

~~~
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": ["arn:aws:s3:::my-bucket/*"]
    }
  ]
}
~~~

Just to be clear, this policy gets assigned directly to the IAM user `my-bucket`. You navigate to IAM > Policies,
and create a new policy. Then go to the Users menu, click on the user's name, click the Permissions tab, and click
the Attach Policy button. Then find the policy you just created in the list and attach it to the user.

The policy I show above is from "Sample 1" at the following link:

[http://blogs.aws.amazon.com/security/post/Tx3VRSWZ6B3SHAV/Writing-IAM-Policies-How-to-grant-access-to-an-Amazon-S3-bucket](http://blogs.aws.amazon.com/security/post/Tx3VRSWZ6B3SHAV/Writing-IAM-Policies-How-to-grant-access-to-an-Amazon-S3-bucket)

After getting this all set up, I tested everything out using the Rails console, and it seemed to be working great.

~~~ ruby
s3 = AWS::S3.new(access_key_id: ENV['AWS_ACCESS_KEY_ID'],
                 secret_access_key: ENV['AWS_SECRET_ACCESS_KEY'])
bucket = s3.buckets['my-bucket']
obj = bucket.objects['new_file.jpg']
obj.write(Pathname.new('/path/to/file.jpg'))
~~~

After running the code above, I was able to successfully see that a new file called `new_file.jpg`
was created in the root of `my-bucket` in my S3 account. Great! This means the connection is working,
so I should be able to upload an image through my website and see it appear on S3.

I gave it a shot, but it didn't work. When I submitted the form to upload my image, I would receive
an `AWS::S3::Errors::AccessDenied` error every time, and the file, of course, did not get uploaded to S3.
Even though my console session and Paperclip were using the same credentials, there was
clearly something different between the way Paperclip was uploading the file and the way I
was doing it from my console session.

I looked closer at the files that were successfully uploaded from my console tests, and I noticed
that they were set as private. In other words, they were not created in a way that they could be
viewed by the public. The whole purpose of using Paperclip and S3 is to host image files that can
be viewed publically so that they can be used on my website. Paperclip must be including an extra
flag when it attempts to upload the file that sets the files permission to be publically readable.

Looking more through the [Paperclip documentation](http://www.rubydoc.info/gems/paperclip/Paperclip/Storage/S3),
specifically the description of the `s3_permissions` option, I see that:

> The default for Paperclip is :public_read.

After a lot more searching, I finally found this list of [IAM Actions for S3](http://docs.aws.amazon.com/IAM/latest/UserGuide/list_s3.html).
On that list, there is an action called `s3:PutObjectAcl`. It turns out that this action is needed
in the IAM policy to ensure that a file can be uploaded that includes a permission setting. Since
Paperclip includes the `:public_read` permission when uploading files, the user must have permission
to set this permission. Finally, with the following IAM policy in place, everything worked as expected:

~~~
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket/*"
            ]
        }
    ]
}
~~~
