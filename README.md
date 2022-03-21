# HubSpot OpsHub & AI Use Case Example
A small use case to show the “Art of the Possible” when it comes to extending HubSpot capabilities using Operations Hub.

## Use Case Example

Imagine there is a support representative handling an open ticket where he is communicating back and forth with the customer through emails and this ticket needs to be escalated to the product team / developers. Instead of having the product team read through the entire emails to understand what the issue is, we can automate the process with Operations Hub plus an AI service that can summarize the text. The end goal here is to have a summarized text of this conversation in which we can fill it into a custom property that we create on the Ticket Object (like “Summary” field), and then we can send this summary to any tool that the product team is using to see the issues.

<img width="1440" alt="Screenshot 2022-03-21 at 22 11 06" src="https://user-images.githubusercontent.com/15332386/159379126-e644d505-7a1f-4f97-9346-fabada7d204b.png">

## High Level Flow

<img width="1218" alt="Screenshot 2022-03-21 at 22 52 11" src="https://user-images.githubusercontent.com/15332386/159379225-43a42803-a494-4434-bc42-fa5bc666f630.png">

## Architecture

<img width="1243" alt="Screenshot 2022-03-21 at 22 24 32" src="https://user-images.githubusercontent.com/15332386/159379288-a1742c5d-f774-40a9-bac4-2936baadcab9.png">

## Worklow Code

```python
import os
import json
from hubspot import HubSpot
import requests
import re
from hubspot.crm.contacts import ApiException

def main(event):

  # How to use secrets
  # Secrets are a way for you to save API keys and set them as a variable to use anywhere in your code
  # Each secret needs to be defined like the example below

  hubspot = HubSpot(api_key=os.getenv('HAPIKEY'))

  try:
    ApiResponse = hubspot.crm.tickets.associations_api.get_all(ticket_id=event.get('inputFields').get('hs_ticket_id'), to_object_type="email")
    result = ApiResponse.results
    
    #getting the id of the last email (last email containts the backtrails of the previous email)
    size = len(result)
    email= result[size-1]
    email_id = email.id 
   
    
  except ApiException as e:
   print(e)
    # We will automatically retry when the code fails because of a rate limiting error from the HubSpot API	.
   raise
    
  url = "https://api.hubapi.com/crm/v3/objects/emails/"+email_id

  querystring = {"properties":["hs_email_text","hs_email_sender_email","hs_email_direction"],"archived":"false","hapikey":os.getenv('HAPIKEY')}

  headers = {'accept': 'application/json'}

  response = requests.request("GET", url, headers=headers, params=querystring)
  
  
  result = response.json()
  print("BEFORE SUMMARY")
  print(result["properties"]["hs_email_text"])
  
  # Formatting the text content of email
  str1 = result["properties"]["hs_email_text"].replace('\n','')
  str2 = str1.replace('>',' ')
  str3  = re.sub('\S*@\S*\s?', '', str2)
  final_str = str3.replace('.','\n')
  
  # Sending text content to the AI service
  deepai_result = requests.post(
    "https://api.deepai.org/api/summarization",
    data={
        'text': final_str,
    },
    headers={'api-key': 'quickstart-QUdJIGlzIGNvbWluZy4uLi4K'})
  summary_text = deepai_result.json()['output']
  print("AFTER SUMMARY")
  print(summary_text)
    

  return {
   "outputFields": {
      "result": summary_text 
    }
  }
```
