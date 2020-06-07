# Azure Blob creation a sing Ansible with Azure Storage REST API
Recently during a client consulting for a client in Ansible Tower we have a requirement to create an Azure blob from the client side, a very restrict envinroment where dependencies are not allowed at this time. We couldn't use Ansible modules like [azure_rm_storageblob](https://docs.ansible.com/ansible/latest/modules/azure_rm_storageblob_module.html) or Azure utils like [azcopy](https://docs.microsoft.com/en-us/azure/storage/common/storage-ref-azcopy-copy) so only option was usage of the [Azure REST API Blob creation](https://docs.microsoft.com/en-us/rest/api/storageservices/put-blob). 

That REST API has some specifics because it requires a SAS (Shared Access Signature) to make the authorization.

The SAS (Shared Access Signature) is created using the Azure Storage Key. We need to set a Authorization header like the below with Storage Account name and Signature but I need to mention that the Signature must be recreated for each request made, that's not fixed value you can set.   
```
Authorization="SharedKey <storage account name>:<signature>"  
```
### SAS 
The SAS (<signature>) is a unique hash-based key which is created for EVERY REST Api execution. Which means we need a key created on the fly during each execution. 
### Headers
We need to set values in the Header like doc (https://docs.microsoft.com/en-us/rest/api/storageservices/put-blob) suggests:
```
Request Headers:  
x-ms-version: 2015-02-21  
x-ms-date: <date>  
Content-Type: text/plain; charset=UTF-8  
x-ms-blob-type: BlockBlob  
Authorization: SharedKey myaccount:YhuFJjN4fAR8/AmBrqBz7MG2uFinQ4rkh4dscbj598g=  
Content-Length: 11 
```
Content-lenght and x-ms-date vary per request and both values must exactly the same between SAS creation and the REST Api Blob creation. 
Plus we have to correctly set the CanonicalizedHeaders and CanonicalizedResource. Here what Wikipedia says about it:
```
In computer science, canonicalization (sometimes standardization or normalization) is a process for converting data that has more than one possible representation into a "standard", "normal", or canonical form. In normal-speak, this means to take the list of items (such as headers in the case of Canonicalized Headers) and standardize them into a required format. Basically, Microsoft decided on a format and you need to match it.
```
We use Python to generate the Shared Access Signature and return the key to the Ansible REST API call in execution-time. That returns the SAS and Request time (time must be the same in the REST API call). Code is below:
```
import requests
import datetime
import hmac
import hashlib
import base64
import sys
storage_account_key = 'TNn8GaTfTUcr20n16BPHvxfo5wLyaxzFXh+kxezc7w+GE5ZtmhhE5nlLGNfK1/Vd/Evnw8iErx/VTEh0It7KPQ=='
container_name='test'
api_version = '2015-02-21'
request_time = datetime.datetime.utcnow().strftime('%a, %d %b %Y %H:%M:%S GMT')
storage_account_name = sys.argv[1]
container_name = sys.argv[2]
blob_name = sys.argv[3]
content_length = sys.argv[4]
string_params = {
    'verb': 'PUT',
    'Content-Encoding': '',
    'Content-Language': '',
    'Content-Length': content_length,
    'Content-MD5': '',
    'Content-Type': 'text/plain; charset=UTF-8',
    'Date': '',
    'If-Modified-Since': '',
    'If-Match': '',
    'If-None-Match': '',
    'If-Unmodified-Since': '',
    'Range': '',
    'CanonicalizedHeaders': 'x-ms-blob-type:BlockBlob' + '\nx-ms-date:' + request_time + '\nx-ms-version:' + api_version + '\n',
    'CanonicalizedResource': '/' + storage_account_name +'/'+container_name+ '/' + blob_name
}

string_to_sign = (string_params['verb'] + '\n' 
                  + string_params['Content-Encoding'] + '\n'
                  + string_params['Content-Language'] + '\n'
                  + string_params['Content-Length'] + '\n'
                  + string_params['Content-MD5'] + '\n' 
                  + string_params['Content-Type'] + '\n' 
                  + string_params['Date'] + '\n' 
                  + string_params['If-Modified-Since'] + '\n'
                  + string_params['If-Match'] + '\n'
                  + string_params['If-None-Match'] + '\n'
                  + string_params['If-Unmodified-Since'] + '\n'
                  + string_params['Range'] + '\n'
                  + string_params['CanonicalizedHeaders']
                  + string_params['CanonicalizedResource'])

signed_string = base64.b64encode(hmac.new(base64.b64decode(storage_account_key), msg=string_to_sign.encode('utf-8'), digestmod=hashlib.sha256).digest()).decode()

print (signed_string+";"+request_time)
```
CanonicalizedHeaders and CanonicalizedResource values must be correctly set with proper new lines characters. In 'string_params' values must be the same of the Blob Creation Api cal  As well as all header params must be appended to 'string_to_sign'. Anything out of the expected is going to fail the REST Api authentication. 
The same SAS procedure/script can be leveraged for any Azure Storage API. I spent a couple hours on this so I hope that helps someone else.

## Ansible playbook to PUT Blob Rest API

Below the Ansible Code which makes the Put Blob operation. Same Header values are required. Blob content should be set in the Request Body.
```
---
- name: Playbook to PUT blob content
  hosts: localhost
  gather_facts: no
  connection: local
  tasks:

  - set_fact:
       storage_account_name: marciotestblob
       blob_container_name: test
       blob_name: myfile.txt
       blob_content: Hello World Blob content

  - name: Generate Shared Access Signature
    shell: python  scripts/generate_key.py {{ storage_account_name }} {{ blob_container_name }} {{ blob_name }} {{ blob_content|length }}
    register: generate_key
    delegate_to: localhost

  - set_fact:
       sas_token: "{{ generate_key.stdout.split(';')[0] }}"
       request_time: "{{ generate_key.stdout.split(';')[1] }}"

  - name: Azure Blob REST API creation
    uri: 
      url: "https://{{ storage_account_name }}.blob.core.windows.net/{{ blob_container_name }}/{{ blob_name }}"
      headers: 
        x-ms-blob-type: BlockBlob
        x-ms-date: "{{ request_time }}"
        x-ms-version: "2015-02-21"
        Content-Type: "text/plain; charset=UTF-8"
        Content-Length: "{{ blob_content|length }}"
        Authorization: "SharedKey {{ storage_account_name }}:{{ sas_token }}"
      method: PUT
      body: "{{ blob_content }}"
      status_code: 201
      validate_certs: false
    delegate_to: localhost
```
In above Ansible code, Blob file details like file name and file content are hardcoded but we can can create dynamic inputs by your requirement.
]
Gitlab repository por this post 