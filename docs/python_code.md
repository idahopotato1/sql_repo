## Python code collection

### 1. Graph API - 

#### create a folder in onedrive for busienss and add a file to it
~~~python
import requests

def create_folder_and_add_excel_file(folder_name, file_name):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    try:
        access_token = session['access_token']
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))

    # Create folder
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/root/children'
    data = {
        'name': folder_name,
        'folder': {}
    }
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/json'
    }
    r = requests.post(endpoint, json=data, headers=headers)
    if r.status_code != 201:
        return "Error creating folder: " + r.text
    folder_id = r.json()['id']

    # Add Excel file
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{folder_id}/children'
    data = {
        'name': file_name,
        'file': {}
    }
    r = requests.post(endpoint, json=data, headers=headers)
    if r.status_code == 201:
        return "Folder and Excel file successfully created"
    else:
        return "Error adding Excel file: " + r.text

~~~

#### find a specific file in onedrive for business and get the file id
~~~python
def get_file_id(file_name):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    try:
        access_token = session['access_token']
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))

    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/root/search(q="{file_name}")'
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/json'
    }
    r = requests.get(endpoint, headers=headers)
    if r.status_code == 200:
        data = r.json()
        for item in data['value']:
            if item['name'] == file_name:
                return item['id']
        return "File not found"
    else:
        return "Error retrieving file ID: " + r.text
~~~

#### modify an excel file in onedrive for business
~~~python
import requests

def modify_excel_file(file_id, sheet_name, cell_address, new_value):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    try:
        access_token = session['access_token']
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))

    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/workbook/worksheets/{sheet_name}/range(address={cell_address})/values'
    data = {
        "values": [[new_value]]
    }
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/json'
    }
    r = requests.patch(endpoint, json=data, headers=headers)
    if r.status_code == 200:
        return "Excel file successfully updated"
    else:
        return "Error updating Excel file: " + r.text


# In this example, the function first checks if the user is logged in by checking the existence of a "user" key in the session object. If the user is not logged in, the function redirects the user to the login page.

# The function then retrieves the access token from the session object, which is required for making authorized requests to the Microsoft Graph API.

# The endpoint URL for updating a specific cell in an Excel worksheet is set in the endpoint variable. The function then sends a PATCH request to the endpoint using the requests library, passing in the new value for the cell in a JSON object. The request includes the access token in the Authorization header, and the Content-Type header set to application/json.

# Finally, the function returns a message indicating whether the Excel file was successfully updated or not.
~~~

#### modify the excel file in onedrive for business to add a dataframe to it
~~~ python
import requests
import pandas as pd

def insert_dataframe_to_excel(file_id, sheet_name, dataframe):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    try:
        access_token = session['access_token']
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))

    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/workbook/worksheets/{sheet_name}/range(address=\'A1\')/values'
    data = {
        "values": dataframe.values.tolist()
    }
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/json'
    }
    r = requests.patch(endpoint, json=data, headers=headers)
    if r.status_code == 200:
        return "DataFrame successfully inserted into Excel file"
    else:
        return "Error inserting DataFrame into Excel file: " + r.text
    ~~~

    #### Now, share this file with somebody
    ~~~python
    import requests

def share_file_with_user(file_id, email):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    try:
        access_token = session['access_token']
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))

    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/invite'
    data = {
        "recipients": [
            {
                "email": email
            }
        ],
        "message": "Please take a look at this file",
        "requireSignIn": True,
        "sendInvitation": True,
        "roles": [
            "write"
        ]
    }
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/json'
    }
    r = requests.post(endpoint, json=data, headers=headers)
    if r.status_code == 201:
        return "File successfully shared with user"
    else:
        return "Error sharing file with user: " + r.text

    ~~~ 

#### Now, check if user has granted permission to send email. If not, ask for permission and then send an email 
~~~python
import requests

def send_email_with_file_link(file_id, recipient_email, subject, message):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    try:
        access_token = session['access_token']
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))

    # Check if the user has granted permission for the Mail.Send scope
    endpoint = 'https://graph.microsoft.com/v1.0/me/mailboxSettings'
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/json'
    }
    r = requests.get(endpoint, headers=headers)
    if r.status_code == 401:
        # The user has not granted permission for the Mail.Send scope
        # Redirect the user to the Microsoft authorization endpoint to request the scope
        return redirect(url_for("authorize", scopes="Mail.Send"))

    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}'
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/json'
    }
    r = requests.get(endpoint, headers=headers)
    if r.status_code != 200:
        return "Error retrieving file information: " + r.text
    file_url = r.json()['webUrl']

    endpoint = 'https://graph.microsoft.com/v1.0/me/sendMail'
    data = {
        "message": {
            "subject": subject,
            "body": {
                "contentType": "Text",
                "content": message + "\n\n" + file_url
            },
            "toRecipients": [
                {
                    "emailAddress": {
                        "address": recipient_email
                    }
                }
            ]
        },
        "saveToSentItems": "false"
    }
    r = requests.post(endpoint, json=data, headers=headers)
    if r.status_code == 202:
        return "Email with file link successfully sent"
    else:
        return "Error sending email with file link: " + r.text
~~~