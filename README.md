Write me 10 API Test Cases for the Below Requirement :



API Verification Protocol : POST , Endpoint :  /user/v1/self/accounts-receivables




AC1: Given all required field values of AR API are provided by level 1 user

When ‘OnBoardingURLRequired’ filed value is ‘1’

Then generate the DMA integration token with iFrame URL

And return success response code of 200




AC2: Given all required field values of AR API are provided by level 1 user

When ‘OnBoardingURLRequired’ filed value is ‘0’

Then generate the DMA integration token WITHOUT iFrame URL

And return success response code of 200




AC3: Given all required field values of AR API are provided by level 1 user

When any of the input field validation fails

Then display error message of “Validation Error”

And return success response code of 400




AC4: Given all required field values of AR API are provided

When level 2 or level 3 users tries to generate the DMA integration token

Then display error message of “Unauthorize”

And return success response code of 401




AC5: Given all required field values of AR API are provided by level 1 user

When tenant is not configured for AR package

Then display error message of “Forbidden”

And return success response code of 403




AC6: Given all required field values of AR API are provided by level 1 user

When Subscriber Not Found or Inactive

Then display error message of “Not Found”

And return success response code of 404




AC7:  Given all required field values of AR API are provided by level 1 user

When environment issue or network failure

Then display error message of “Internal Server Error”

And return success response code of 500




AC8: All relevant information needs to be logged in Splunk




AC9: UVM Scans needs to be done for the AR Onboarding API and no new issues should be introduced as part of this change




UI Verifications:

AC10: Given Melio/ FI/Non-FI application

When user level 1 logs-in

Then allow it to successfully enroll for Accounts Receivables packages



Note :

AC means Acceptance criteria
AR menas Account Recievables 
Melio >>> mentioned here is a Payment Engine used for Buisness payments in US 
