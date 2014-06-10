# Quickbooks Web Connector (QBWC)

Be Warned, this code is still hot out of the oven. 

## Installation

Install the gem

  `gem install qbwc`

Add it to your Gemfile

  `gem "qbwc"`

Run the generator:

  `rails generate qbwc:install`

## Features

QBWC was designed to add quickbooks web connector integration to your Rails 3 application. 

* Implementation of the Soap WDSL spec for Intuit Quickbooks and Point of Sale
* Integration with the [qbxml](https://github.com/skryl/qbxml) gem providing qbxml processing

## Getting Started

### Configuration

All configuration takes place in the gem initializer. See the initializer for more details regarding the configuration options.

### Basics

The QBWC gem provides a persistent work queue for the Web Connector to talk to.

Every time the Web Connector initiates a new conversation with the application a
Session will be created. The Session is a collection of jobs and the requests
that comprise these jobs. A new Session will automatically queue up all the work
available across all currently enabled jobs for processing by the web connector.
The session instance will persist across all requests until the work it contains
has been exhausted. You never have to interact with the Session class directly
(unless you want to...) since creating a new job will automatically add it's
work to the next session instance.

A Job is just a named work queue. It consists of a name, a company (defaults to QBWC.company_file_path), and some qbxml requests. If requests are not provided, a code block that generates next qbxml request can be provided.

*Note: All requests may be in ruby hash form, generated qbxml
Raw requests are supported supported as of 0.0.3 (8/28/2012)*

The code block is called every time a session must send a request. If block return nil, no request will be send and next pending job will be checked.

Only enabled jobs with pending requests are added to a new session instance. Pending requests is checked calling code block, but an optional pending requests checking block can also be added to a job, so request creation can be avoided.

An optional response processor block can also be added to a job. Responses to
all requests are processed immediately after being received.

Here is the rough order in which things happen:

  1. The Web Connector initiates a connection
  2. A new Session is created (with work from all enabled jobs with pending requests)
  3. The web connector requests work
  4. The session responds with the next request in the work queue
  5. The web connector provides a response
  6. The session responds with the current progress of the work queue (0 - 100)
  6. The response is processed
  7. If progress == 100 then the web connector closes the connection, otherwise goto 3

### Adding Jobs

Create a new job

    QBWC.add_job('my job') do
      # work to do
    end

Add a checking proc

    QBWC.jobs['my job'].set_checking_proc do
      # pending requests checking here
    end

Add a response proc

    QBWC.jobs['my job'].set_response_proc do |r|
      # response processing work here
    end

Caveats
  * Jobs are enabled by default
  * Using a non unique job name will overwrite the existing job

###Sample Jobs

Add a Customer (Wrapped)

          {  :qbxml_msgs_rq => 
            [
              {
                :xml_attributes =>  { "onError" => "stopOnError"}, 
                :customer_add_rq => 
                [
                  {
                    :xml_attributes => {"requestID" => "1"},  ##Optional
                    :customer_add   => { :name => "GermanGR" }
                  } 
                ] 
              }
            ]
          }
          
Add a Customer (Unwrapped)

        {
          :customer_add_rq    => 
          [
            {
              :xml_attributes => {"requestID" => "1"},  ##Optional
              :customer_add   => { :name => "GermanGR" }
            } 
          ] 
        }

Get All Vendors (In Chunks of 5)

        QBWC.add_job(:import_vendors, nil
          {
            :vendor_query_rq  =>
            {
              :xml_attributes => { "requestID" =>"1", 'iterator'  => "Start" },
      
              :max_returned => 5,
              :owner_id => 0,
              :from_modified_date=> "1984-01-29T22:03:19"

            }
          }
        )
        
Get All Vendors (Raw QBXML)

        QBWC.add_job(:import_vendors, nil
          '<?xml version="1.0"?>
          <?qbxml version="7.0"?>
          <QBXML>
            <QBXMLMsgsRq onError="continueOnError">
            <VendorQueryRq requestID="6" iterator="Start">
            <MaxReturned>5</MaxReturned>
            <FromModifiedDate>1984-01-29T22:03:19-05:00</FromModifiedDate>
            <OwnerID>0</OwnerID>
          </VendorQueryRq>
          </QBXMLMsgsRq>
          </QBXML>
          '
        )

### Managing Jobs

Jobs can be added, removed, enabled, and disabled. See the above section for
details on adding new jobs. 

Removing jobs is as easy as deleting them from the jobs hash.                   

    QBWC.jobs.delete('my job')

Disabling a job

    QBWC.jobs['my job'].disable

Enabling a job

    QBWC.jobs['my job'].enable

### Supporting multiple users/companies

Override get_user and current_company methods in the generated controller. authenticate_user must authenticate with username and password and return user if it's authenticated, nil in other case. current_company receives authenticated user and must return nil if there are no pending jobs or company where jobs will run. Currently this methods are like this:

    protected
    def authenticate_user(username, password)
      username if username == QBWC.username && password == QBWC.password
    end
    def current_company(user)
      QBWC.company_file_path if QBWC.pending_jobs(QBWC.company_file_path).present?
    end


### Check versions ###

If you want to return server version or check client version you can override server_version_response or check_client_version methods in your controller. Check QB web connector guide for allowed responses.

## Contributing to qbwc
 
* Check out the latest master to make sure the feature hasn't been implemented or the bug hasn't been fixed yet
* Check out the issue tracker to make sure someone already hasn't requested it and/or contributed it
* Fork the project
* Start a feature/bugfix branch
* Commit and push until you are happy with your contribution
* Make sure to add tests for it. This is important so I don't break it in a future version unintentionally.
* Please try not to mess with the Rakefile, version, or history. If you want to have your own version, or is otherwise necessary, that is fine, but please isolate to its own commit so I can cherry-pick around it.
