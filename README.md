# Inferno Template

This is a template repository for an
[Inferno](https://github.com/inferno-community/inferno-core) test kit.

## Documentation
- [Inferno documentation](https://inferno-framework.github.io/inferno-core/)
- [Ruby API documentation](https://inferno-framework.github.io/inferno-core/docs)
- [JSON API documentation](https://inferno-framework.github.io/inferno-core/api-docs)

## Instructions for Developing tests

- Clone this repo/Click "Use this template" on github.
- Run `setup.sh` in this repo to pull the needed docker images and set up the
  database.
- Put the `package.tgz` for the IG you're writing tests for in
  `lib/your_test_kit_name/igs` and update this path in
  `docker-compose.background.yml`.
  This will ensure that the validator has access to the resources needed to
  validate resources against your IG.
  
All tests you write should go in the `lib` folder, which currently contains some
example tests. Refer to the [Inferno
documentation](https://inferno-framework.github.io/inferno-core/) for more
information about the specifics of writing tests.

### Development with Docker vs Docker + ruby
Tests can be developed with or without a local ruby installation using docker.
However, it takes much longer to build/start/stop docker than to start/stop
native ruby processes. Inferno needs to be restarted whenever tests change, so
you will be starting and stopping it often during test development. For this
reason, it is highly recommended that you install ruby and do not rely solely on
docker for development if possible.

### Development with Docker only
- Run `run.sh` to build your tests and run inferno. You will need to stop and
  re-run this whenever you make changes to your tests.
- Navigate to `http://localhost` to access Inferno, where your test suite will
  be available. To access the FHIR resource validator, navigate to
  `http://localhost/validator`.

### Development with ruby
It is highly recommended that you install ruby via a [ruby version
manager](https://www.ruby-lang.org/en/documentation/installation/#managers). When you

- Run `bundle install` to install dependencies.
- Run `bundle exec inferno migrate` to set up the database.
- Run `gem install foreman` to install foreman, which will be used to run the
  Inferno web and worker processes.
- Run `bundle exec inferno services start` to start the background services. By
  default, these include nginx, redis, the FHIR validator service, and the FHIR
  validator UI. Background services can be added/removed/edited in
  `docker-compose.background.yml`.
- Run `inferno start` to start Inferno. You will need to stop and re-run this
  whenever you make changes to your tests.
- Navigate to `http://localhost:4567` to access Inferno, where your test suite will
  be available. To access the FHIR resource validator, navigate to
  `http://localhost/validator`.
- When you are done, run `bundle exec inferno services stop` to stop the
  background services.

#### Interactive consoles
A local ruby installation also allows you to use [pry](https://pry.github.io/),
a powerful interactive console to explore and experiment with your tests with
`inferno console`:
```ruby
ᐅ bundle exec inferno console
[1] pry(main)> suite = InfernoTemplate::Suite
=> InfernoTemplate::Suite
[2] pry(main)> suite.groups
=> [#<Class:0x00000001539ecd30>, #<Class:0x0000000155509500>]
[3] pry(main)> suite.groups.map(&:title)
=> ["Capability Statement", "Patient  Tests"]
[4] pry(main)> suite.groups.first.tests.map(&:title)
=> ["Read CapabilityStatement"]
```

It is also possible to set a breakpoint using the [debug
gem](https://github.com/ruby/debug) within a test's `run` block to debug test
behavior:
- Add `require 'debug/open_nonstop'` and `debugger` to set the breakpoint.
- Run your tests until the breakpoint is reached.
- In a separate terminal window, run `bundle exec rdbg -A` to access the
  interactive console.

```ruby
module InfernoTemplate
  class PatientGroup < Inferno::TestGroup
    ...
    test do
      ...
      run do
        fhir_read(:patient, patient_id, name: :patient)

        require 'debug/open_nonstop'
        debugger

        assert_response_status(200)
        assert_resource_type(:patient)
        assert resource.id == patient_id,
               "Requested resource with id #{patient_id}, received resource with id #{resource.id}"
      end
    end
  end
end
```

```ruby
ᐅ bundle exec rdbg -A
DEBUGGER (client): Connected. PID:22112, $0:sidekiq 6.5.7  [0 of 10 busy]

[18, 27] in ~/code/inferno-template/lib/inferno_template/patient_group.rb
    18|
    19|       run do
    20|         fhir_read(:patient, patient_id, name: :patient)
    21|
    22|         require 'debug/open_nonstop'
=>  23|         debugger
    24|
    25|         assert_response_status(200)
    26|         assert_resource_type(:patient)
    27|         assert resource.id == patient_id,
(ruby:remote) self.id
"test_suite_template-patient_group-Test01"
(ruby:remote) self.title
"Server returns requested Patient resource from the Patient read interaction"
(rdbg:remote) inputs
[:patient_id, :url, :credentials]
(ruby:remote) patient_id
"85"
(rdbg:remote) url
"https://inferno.healthit.gov/reference-server/r4"
(rdbg:remote) ls request    # outline command
Inferno::Entities::Request#methods:
  created_at        created_at=  direction     direction=     headers          headers=          id        id=         index          index=          name             name=
  query_parameters  request      request_body  request_body=  request_header   request_headers   resource  response    response_body  response_body=  response_header  response_headers
  result_id         result_id=   status        status=        test_session_id  test_session_id=  to_hash   updated_at  updated_at=    url             url=             verb
  verb=
instance variables: @created_at  @direction  @headers  @id  @index  @name  @request_body  @response_body  @result_id  @status  @test_session_id  @updated_at  @url  @verb
(ruby:remote) request.status
200
(ruby:remote) request.response_body
"{\n  \"resourceType\": \"Patient\" ... }"
(rdbg:remote) ?    # help command

### Control flow

* `s[tep]`
  * Step in. Resume the program until next breakable point.
...
```

## Distributing tests

In order to make your test suite available to others, it needs to be organized
like a standard ruby gem (ruby libraries are called gems).

- Fill in the information in the `gemspec` file in the root of this repository.
  The name of this file should match the `spec.name` within the file. This will
  be the name of the gem you create. For example, if your file is
  `my_test_kit.gemspec` and its `spec.name` is `'my_test_kit'`, then others will
  be able to install your gem with `gem install my_test_kit`. There are
  [recommended naming conventions for
  gems](https://guides.rubygems.org/name-your-gem/).
- Your tests must be in `lib`
- `lib` should contain only one file. All other files should be in a
  subdirectory. The file in lib be what people use to import your gem after they
  have installed it. For example, if your test kit contains a file
  `lib/my_test_suite.rb`, then after installing your test kit gem, I could
  include your test suite with `require 'my_test_suite'`.
- **Optional:** Once your gemspec file has been updated, you can publish your
  gem on [rubygems, the official ruby gem repository](https://rubygems.org/). If
  you don't publish your gem on rubygems, users will still be able to install it
  if it is located in a public git repository. To publish your gem on rubygems,
  you will first need to [make an account on
  rubygems](https://guides.rubygems.org/publishing/#publishing-to-rubygemsorg)
  and then run `gem build *.gemspec` and `gem push *.gem`.

## Example Inferno test kits

- https://github.com/inferno-community/ips-test-kit
- https://github.com/inferno-community/shc-vaccination-test-kit

## License
Copyright 2022 The MITRE Corporation

Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License at
```
http://www.apache.org/licenses/LICENSE-2.0
```
Unless required by applicable law or agreed to in writing, software distributed
under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR
CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.
