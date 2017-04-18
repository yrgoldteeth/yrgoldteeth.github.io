---
layout: post
title: Using sequel-rails With A Forking Webserver
---
I was contracted recently for a rush job on a dashboard-style reporting
webapp using a postgres database whose schema I did not design or
control.  I had long intended on looking into the [Sequel][sequel] 
library for database interactions, and as I was going to be developing
the application in Rails, I was happy to see that there was also an
existing [sequel-rails][sequel-rails] gem that works almost exactly as
expected similar to the stock ActiveRecord style of Rails apps.  

I was able to easily make changes to the database in a migration style,
and the application in general just went off without a hitch.

The application needed a staging/demo server that could be provisioned
quickly, so I went with Heroku in a manner similar to the instructions
in this [article][heroku-deploy], which calls for the [Unicorn][unicorn]
webserver with a configuration like so:

    # config/unicorn.rb
    worker_processes Integer(ENV["WEB_CONCURRENCY"] || 3)
    timeout 15
    preload_app true

    before_fork do |server, worker|
      Signal.trap 'TERM' do
        puts 'Unicorn master intercepting TERM and sending myself QUIT instead'
        Process.kill 'QUIT', Process.pid
      end

      defined?(ActiveRecord::Base) and
        ActiveRecord::Base.connection.disconnect!
    end

    after_fork do |server, worker|
      Signal.trap 'TERM' do
        puts 'Unicorn worker intercepting TERM and doing nothing. Wait for master to send QUIT'
      end

      defined?(ActiveRecord::Base) and
        ActiveRecord::Base.establish_connection
    end

As the application does not use ActiveRecord, I removed those lines and
gave the deploy a shot, knowing it would likely become a problem.  Soon, 
it was manifesting itself by throwing 500 errors when a user logged in,
but would clear up on a refresh.  This irritated me, though, but
everything I found on the internet about using [Sequel][sequel] with a
forking webserver referred to reconnecting just using the DB object that
was always just created globally in the application, which is not how
[sequel-rails][sequel-rails] does things.  So, after digging through the
code for a while I found that the DB object when using sequel-rails is
at `Sequel::Model.db`, which allowed me to clear up the 500 errors by
making this adjustment to the config file: 

    # config/unicorn.rb
    worker_processes Integer(ENV["WEB_CONCURRENCY"] || 3)
    timeout 15
    preload_app true

    before_fork do |server, worker|
      Signal.trap 'TERM' do
        puts 'Unicorn master intercepting TERM and sending myself QUIT instead'
        Process.kill 'QUIT', Process.pid
      end

      defined?(Sequel::Model) and
        Sequel::Model.db.disconnect
    end

    after_fork do |server, worker|
      Signal.trap 'TERM' do
        puts 'Unicorn worker intercepting TERM and doing nothing. Wait for master to send QUIT'
      end
      
      defined?(Sequel::Model) and
        Sequel::Model.db.connect(SequelRails.configuration.environment_for(Rails.env))
    end

I hope that this saves some headaches for developers down the line.

[heroku-deploy]: https://devcenter.heroku.com/articles/rails-unicorn
[sequel-rails]: https://github.com/TalentBox/sequel-rails
[sequel]: https://github.com/jeremyevans/sequel
[unicorn]: http://unicorn.bogomips.org/
