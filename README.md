# Problem
Today every competing JavaScript functions API is different enough that we end up with lower developer productivity and vendor lock-in. Lower developer productivity because developers have to learn multiple ways to write the JavaScript function code and may have to write more complex code to avoid vendor lock-in. Organizations wanting to leverage multiple functions providers or move from one provider to another incur significant additional cost.
# Goal
Goal is to define a JavaScript functions API that avoids vendor lock-in and facilitates developer productivity while being general enough to allow different implementations and optimizations behind the scenes.

Things we should try to standardize
* function sig (including parameters and what’s available on them)
  * how functions get events(CloudEvents)?
* key supporting apis like
  * log method signature
  * exporting the function
  * how to report an error

Things we should not try to standardize
* JS framework for invoking the functions
  * http framework that is used(Fastify, Express, etc…)
* underlying buildpack/image that ends up running
* accessing platform specific APIs offered by the vendor platforms(ex: google apis)
* output format/how you view logs
* how you monitor functions
# Scenarios

Scenario 1: Writing a HelloWorld function in one vendors platform and moving to other vendors platforms
Shouldn’t have to change the code
Build steps and or configuration may have to change

Scenario 2: Writing in one vendor and moving to another (ex using Google Cloud Functions and move to Cloudflare workers)
If code does not use vendor specific APIs you should not have to change the code.
If code uses vendor specific APIs, you may need to change the code if you also need to use a different vendor for those calls.
Build steps and or configuration may have to change

