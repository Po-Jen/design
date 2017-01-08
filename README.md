# ROS 2.0 design

原始的repository是一個[Jekyll](http://jekyllrb.com/)網站，架設在[Github Pages](http://pages.github.com/)上。 http://design.ros2.org/

出於對ROS 2.0的興趣，同時也想推廣，我啟動了中文化的網站，如果你有興趣，歡迎來看看。http://po-jen.github.io/design/

The best mailing list for discussing these topics is [ros-sig-ng-ros@googlegroups.com](mailto:ros-sig-ng-ros@googlegroups.com).
You can view the archives [here](https://groups.google.com/forum/?fromgroups#!forum/ros-sig-ng-ros)

## Working Locally

You can run the site locally by running this command in this repository:

```
jekyll serve --watch --baseurl=''
```

And navgiating to your browser to:

[http://localhost:4000/](http://localhost:4000/)


## Site Setup

Site is a Jekyll website with `design.ros2.org` as the `CNAME`.

The site requires no static generation outside of github's static jekyll generation, which means that changes are published to the site as soon as they are pushed (can take up to 10 minutes for github to update the site).

The github login (for showing pull requests) also requires that https://github.com/prose/gatekeeper is setup in heroku, and the url for that is http://auth.design.ros2.org/authenticate/TEMP_TOKEN. Because of the free Heroku instance, the first time someone logins in after a period of time, there is a delay.
