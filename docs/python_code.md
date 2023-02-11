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

#### use openpyxl to modify the excel file and save it back to sharepoint
~~~python
import requests
import openpyxl

def modify_excel_file_in_onedrive(file_id, dataframe):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    try:
        access_token = session['access_token']
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))

    # Download the file from OneDrive
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/content'
    headers = {
        'Authorization': 'Bearer ' + access_token
    }
    r = requests.get(endpoint, headers=headers)
    if r.status_code != 200:
        return "Error downloading file: " + r.text
    with open("temp.xlsx", "wb") as f:
        f.write(r.content)

    # Modify the file using openpyxl
    add_dataframe_to_excel_file("temp.xlsx", dataframe)

    # Upload the modified file back to OneDrive
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/content'
    with open("temp.xlsx", "rb") as f:
        data = f.read()
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/octet-stream'
    }
    r = requests.put(endpoint, headers=headers, data=data)
    if r.status_code == 200:
        return "File successfully modified"
    else:
        return "Error modifying file: " + r.text

~~~

#### now, copy a file from onedrive to a new location and then share it wit someone. 
~~~python
import requests

def copy_file_and_share(file_id, folder_name, recipient_email, subject, message):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    if "Files.ReadWrite" not in session.get("scopes", []):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    try:
        access_token = session['access_token']
        refresh_token = session["refresh_token"]
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))

    # Create a new folder in OneDrive
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/root/children'
    data = {'name': folder_name, 'folder': {}}
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/json'
    }
    r = requests.post(endpoint, json=data, headers=headers)
    if r.status_code != 201:
        return "Error creating folder: " + r.text
    folder_id = r.json()['id']

    # Copy the file to the new folder
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/copy'
    data = {'parentReference': {'id': folder_id}}
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/json'
    }
    r = requests.post(endpoint, json=data, headers=headers)
    if r.status_code != 200:
        return "Error copying file: " + r.text
    copied_file_id = r.json()['id']

    # Share the copied file with the recipient
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{copied_file_id}/invite'
    data = {
        "requireSignIn": False,
        "sendInvitation": True,
        "recipients": [
            {
                "email": recipient_email,
                "alias": recipient_email
            }
        ],
        "message": message
    }
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/json'
    }
    r = requests.post(endpoint, json=data, headers=headers)
    if r.status_code != 202:
        return "Error sharing file: " + r.text

    return "File successfully copied and shared"

~~~ 

#### use openpyxl to add a chart to the excel and save it back to onedrive
~~~python
import requests
import io
import openpyxl

def modify_excel_file_in_onedrive(file_id):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    if "Files.ReadWrite" not in session.get("scopes", []):
        session['next_url'] = request.path
        return redirect(url_for("login"))

    try:
        access_token = session['access_token']
        refresh_token = session["refresh_token"]
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))

    # Download the file from OneDrive
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/content'
    headers = {'Authorization': 'Bearer ' + access_token}
    r = requests.get(endpoint, headers=headers)
    if r.status_code != 200:
        return "Error downloading file: " + r.text
    file = io.BytesIO(r.content)

    # Modify the file using openpyxl
    wb = openpyxl.load_workbook(file)
    # Add code to modify the file using openpyxl here...
    file.seek(0)
    wb.save(file)

    # Upload the modified file back to OneDrive
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/content'
    headers = {
        'Authorization': 'Bearer ' + access_token,
        'Content-Type': 'application/octet-stream'
    }
    r = requests.put(endpoint, headers=headers, data=file.read())
    if r.status_code != 200:
        return "Error uploading file: " + r.text

    return "File successfully modified and uploaded"

~~~

#### now, let's add all the functions together to create an app to create a folder in onedrive, copy a file from onedrive and paste it in the folder that was just created. and then, add a dataframe into the file. use the openpyxl to format the dataframe. and finally, share the file with somebody. we will also need to check if scope has been granted
~~~python
import requests
import io
import openpyxl
import pandas as pd

def create_folder_in_onedrive(folder_name):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))
    if "Files.ReadWrite" not in session.get("scopes", []):
        session['next_url'] = request.path
        return redirect(url_for("login"))
    try:
        access_token = session['access_token']
        refresh_token = session["refresh_token"]
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/root/children'
    try:
        r = requests.get(endpoint,
                         headers={'Authorization': 'Bearer ' + access_token})
        data = {'name': folder_name, 'folder': {}, '@microsoft.graph.conflictBehavior': 'rename'}
        r = requests.post(endpoint, json=data,
                            headers={'Authorization': 'Bearer ' + access_token})
        return r.json()["id"]
    except:
        return "Error creating folder"

def copy_file_in_onedrive(file_id, folder_id):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))
    if "Files.ReadWrite" not in session.get("scopes", []):
        session['next_url'] = request.path
        return redirect(url_for("login"))
    try:
        access_token = session['access_token']
        refresh_token = session["refresh_token"]
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/copy'
    data = {"parentReference": {"id": folder_id}}
    try:
        r = requests.post(endpoint, json=data, headers={'Authorization': 'Bearer ' + access_token})
        return r.json()["id"]
    except:
        return "Error copying file"

def modify_excel_file_in_onedrive(file_id):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))
    if "Files.ReadWrite" not in session.get("scopes", []):
        session['next_url'] = request.path
        return redirect(url_for("login"))
    try:
        access_token = session['access_token']
        refresh_token = session["refresh_token"]
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))

    # Download the file from OneDrive
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/content'
    r = requests.get(endpoint, headers={'Authorization': 'Bearer ' + access_token})
    file = io.BytesIO(r.content)

    # Load the file into a pandas dataframe
    df = pd.DataFrame({'A': [1, 2, 3], 'B': [4, 5, 6], 'C': [7, 8, 9]})

    # Write the dataframe to the excel file using openpyxl
    book = openpyxl.load_workbook(file)
    writer = pd.ExcelWriter(file, engine='openpyxl') 
    writer.book = book
    df.to_excel(writer, index=False, sheet_name='Sheet1')
    writer.save()

    # Upload the modified file back to OneDrive
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/content'
    file.seek(0)
    r = requests.put(endpoint, data=file.read(), headers={'Authorization': 'Bearer ' + access_token,
                                                           'Content-Type': 'application/octet-stream'})
    return "File modified successfully"

def share_file_in_onedrive(file_id, email):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))
    if "Mail.Send" not in session.get("scopes", []):
        session['next_url'] = request.path
        return redirect(url_for("login"))
    try:
        access_token = session['access_token']
        refresh_token = session["refresh_token"]
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/invite'
    data = {"recipients": [{"emailAddress": {"address": email}}],
            "message": "Please check this file",
            "requireSignIn": True,
            "sendInvitation": True,
            "roles": ["read"]}
    try:
        r = requests.post(endpoint, json=data, headers={'Authorization': 'Bearer ' + access_token})
        return "File shared successfully"
    except:
        return "Error sharing file"

def check_and_retrieve_modified_file(file_id):
    if not session.get("user"):
        session['next_url'] = request.path
        return redirect(url_for("login"))
    if "Files.ReadWrite" not in session.get("scopes", []):
        session['next_url'] = request.path
        return redirect(url_for("login"))
    try:
        access_token = session['access_token']
        refresh_token = session["refresh_token"]
    except:
        session['next_url'] = request.path
        return redirect(url_for("login"))
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/content'
    r = requests.get(endpoint, headers={'Authorization': 'Bearer ' + access_token})
    file = io.BytesIO(r.content)

    # Load the data from the file into a pandas dataframe
    df = pd.read_excel(file)
    return df

# Example usage
folder_id = create_folder_in_onedrive("Example Folder")
file_id = copy_file_in_onedrive("FILE_ID", folder_id)
modify_excel_file_in_onedrive(file_id)
share_file_in_onedrive(file_id, "eddie.wy.zhu@gmail.com")
df = check_and_retrieve_modified_file(file_id)
print(df)
~~~

#### create an eletron app to do that 
~~~python 
from flask import Flask, request, jsonify
import openpyxl
import pandas as pd
import requests

app = Flask(__name__)

@app.route('/create_folder', methods=['POST'])
def create_folder_in_onedrive():
    data = request.get_json()
    foldername = data['folder_name']
    access_token = data['access_token']

    endpoint = 'https://graph.microsoft.com/v1.0/me/drive/root/children'
    r = requests.get(endpoint, headers={'Authorization': 'Bearer ' + access_token})
    data = {'name': foldername, 'folder': {}, '@microsoft.graph.conflictBehavior': 'rename'}
    r = requests.post(endpoint, json=data, headers={'Authorization': 'Bearer ' + access_token})
    return jsonify({'response': r.json()})

@app.route('/copy_file', methods=['POST'])
def copy_file_in_onedrive():
    data = request.get_json()
    file_id = data['file_id']
    folder_id = data['folder_id']
    access_token = data['access_token']

    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/copy'
    data = {'parentReference': {'id': folder_id}}
    r = requests.post(endpoint, json=data, headers={'Authorization': 'Bearer ' + access_token})
    return jsonify({'response': r.json()})

@app.route('/modify_file', methods=['POST'])
def modify_excel_file_in_onedrive():
    data = request.get_json()
    file_id = data['file_id']
    access_token = data['access_token']

    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/content'
    r = requests.get(endpoint, headers={'Authorization': 'Bearer ' + access_token}, stream=True)
    df = pd.read_excel(r.raw)
    df['new_column'] = 'new_value'
    buffer = io.BytesIO()
    writer = pd.ExcelWriter(buffer, engine='openpyxl')
    book = openpyxl.load_workbook(r.raw)
    writer.book = book
    df.to_excel(writer, index=False, sheet_name='Sheet1')
    writer.save()
    file = buffer.getvalue()
    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/content'
    r = requests.put(endpoint, data=file, headers={'Authorization': 'Bearer ' + access_token, 'Content-Type':
    'Content-Length': len(file)})
    return jsonify({'response': r.json()})

@app.route('/share_file', methods=['POST'])
def share_file_in_onedrive():
    data = request.get_json()
    file_id = data['file_id']
    email = data['email']
    access_token = data['access_token']

    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/invite'
    data = {
        'recipients': [
            {
                'email': email
            }
        ],
        'requireSignIn': False,
        'sendInvitation': True,
        'roles': [
            'write'
        ]
    }
    r = requests.post(endpoint, json=data, headers={'Authorization': 'Bearer ' + access_token})
    return jsonify({'response': r.json()})

@app.route('/check_file', methods=['POST'])
def check_and_retrieve_modified_file():
    data = request.get_json()
    file_id = data['file_id']
    access_token = data['access_token']

    endpoint = f'https://graph.microsoft.com/v1.0/me/drive/items/{file_id}/content'
    r = requests.get(endpoint, headers={'Authorization': 'Bearer ' + access_token}, stream=True)
    df = pd.read_excel(r.raw)
    return jsonify({'data': df.to_dict()})

if __name__ == '__main__':
    app.run()
~~~
#### frontend
~~~javascript
const {app, BrowserWindow, ipcMain} = require('electron')
const path = require('path')
const url = require('url')

let mainWindow

function createWindow () {
  mainWindow = new BrowserWindow({width: 800, height: 600})
  mainWindow.loadURL(url.format({
    pathname: path.join(__dirname, 'index.html'),
    protocol: 'file:',
    slashes: true
  }))
  mainWindow.on('closed', function () {
    mainWindow = null
  })
}

app.on('ready', createWindow)

app.on('window-all-closed', function () {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})

app.on('activate', function () {
  if (mainWindow === null) {
    createWindow()
  }
})

ipcMain.on('create_folder', (event, folder_name, access_token) => {
  fetch('http://localhost:5000/create_folder', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      folder_name: folder_name,
      access_token: access_token
    })
  })
  .then(res => res.json())
  .then(json => {
    event.sender.send('create_folder_response', json)
  })
})

ipcMain.on('copy_file', (event, file_id, folder_id, access_token) => {
  fetch('http://localhost:5000/copy_file', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      file_id: file_id,
      folder_id: folder_id,
      access_token: access_token
    })
  })
  .then(res => res.json())
  .then(json => {
    event.sender.send('copy_file_response', json)
  })
})

ipcMain.on('modify_file', (event, file_id, access_token) => {
  fetch('http://localhost:5000/modify_file', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      file_id: file_id,
      access_token: access_token
    })
  })
  .then(res => res.json())
  .then(json => {
    event.sender.send('modify_file_response', json)
  })
})

ipcMain.on('share_file', (event, file_id, email, access_token) => {
  fetch('http://localhost:5000/share_file', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      file_id: file_id,
     
      email: email,
      access_token: access_token
    })
  })
  .then(res => res.json())
  .then(json => {
    event.sender.send('share_file_response', json)
  })
})

ipcMain.on('check_file', (event, file_id, access_token) => {
  fetch('http://localhost:5000/check_file', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      file_id: file_id,
      access_token: access_token
    })
  })
  .then(res => res.json())
  .then(json => {
    event.sender.send('check_file_response', json)
  })
})
~~~ 
##### html file
~~~html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>OneDrive App</title>
  </head>
  <body>
    <h1>Create Folder</h1>
    <form>
      <input type="text" id="folder_name" placeholder="Folder Name">
      <input type="text" id="access_token" placeholder="Access Token">
      <button id="create_folder_button">Create Folder</button>
    </form>
    <br><br>
    <h1>Copy File</h1>
    <form>
      <input type="text" id="file_id" placeholder="File ID">
      <input type="text" id="folder_id" placeholder="Folder ID">
      <input type="text" id="access_token" placeholder="Access Token">
      <button id="copy_file_button">Copy File</button>
    </form>
    <br><br>
    <h1>Modify File</h1>
    <form>
      <input type="text" id="file_id" placeholder="File ID">
      <input type="text" id="access_token" placeholder="Access Token">
      <button id="modify_file_button">Modify File</button>
    </form>
    <br><br>
    <h1>Share File</h1>
    <form>
      <input type="text" id="file_id" placeholder="File ID">
      <input type="text" id="email" placeholder="Email">
      <input type="text" id="access_token" placeholder="Access Token">
      <button id="share_file_button">Share File</button>
    </form>
    <br><br>
    <h1>Check File</h1>
    <form>
      <input type="text" id="file_id" placeholder="File ID">
      <input type="text" id="access_token" placeholder="Access Token">
      <button id="check_file_button">Check File</button>
    </form>
    <script>
      const {ipcRenderer} = require('electron')
      const createFolderButton = document.getElementById('create_folder_button')
      const copyFileButton = document.getElementById('copy_file_button')
      const modifyFileButton = document.getElementById('modify_file_button')
      const shareFileButton = document.getElementById('share_file_button')
      const checkFileButton = document.getElementById('check_file_button')
      createFolderButton.addEventListener('click', (event) => {
        event.preventDefault()
        const folderName = document.getElementById('folder_name').value
        const accessToken = document.getElementById('access_token').value
        ipcRenderer.send('create_folder', folderName, accessToken)
      })
      copyFileButton.addEventListener('click', (event) => {
        event.preventDefault()
        const fileId = document.getElementById('file_id').value
        const folderId = document.getElementById('folder_id').value
                const accessToken = document.getElementById('access_token').value
        ipcRenderer.send('copy_file', fileId, folderId, accessToken)
      })
      modifyFileButton.addEventListener('click', (event) => {
        event.preventDefault()
        const fileId = document.getElementById('file_id').value
        const accessToken = document.getElementById('access_token').value
        ipcRenderer.send('modify_file', fileId, accessToken)
      })
      shareFileButton.addEventListener('click', (event) => {
        event.preventDefault()
        const fileId = document.getElementById('file_id').value
        const email = document.getElementById('email').value
        const accessToken = document.getElementById('access_token').value
        ipcRenderer.send('share_file', fileId, email, accessToken)
      })
      checkFileButton.addEventListener('click', (event) => {
        event.preventDefault()
        const fileId = document.getElementById('file_id').value
        const accessToken = document.getElementById('access_token').value
        ipcRenderer.send('check_file', fileId, accessToken)
      })
      ipcRenderer.on('create_folder_response', (event, arg) => {
        console.log(arg)
      })
      ipcRenderer.on('copy_file_response', (event, arg) => {
        console.log(arg)
      })
      ipcRenderer.on('modify_file_response', (event, arg) => {
        console.log(arg)
      })
      ipcRenderer.on('share_file_response', (event, arg) => {
        console.log(arg)
      })
      ipcRenderer.on('check_file_response', (event, arg) => {
        console.log(arg)
      })
    </script>
  </body>
</html>
~~~ 
##### script to start flask and electron in the package single app - start-app.sh
~~~bash
#!/usr/bin/env bash

# Start the Flask backend
FLASK_APP=app.py FLASK_ENV=development flask run &

# Store the PID of the Flask backend
FLASK_PID=$!

# Start the Electron frontend
electron .

# Kill the Flask backend when Electron exits
trap "kill $FLASK_PID" EXIT
~~~

##### forge to package the app
~~~json
{
  "name": "onedrive-app",
  "productName": "OneDrive App",
  "version": "1.0.0",
  "description": "OneDrive App",
  "main": "main.js",
  "scripts": {
    "start": "electron-forge start",
    "package": "electron-forge package",
    "make": "electron-forge make",
    "publish": "electron-forge publish",
    "lint": "eslint .",
    "dist": "electron-forge make --platform=win32 --arch=x64"
  },
  "repository": {
    "type": "git",
    "url": "..."
    },
    "keywords": [
        "electron",
        "electron-forge",
        "react",
        "webpack"
        ],
        "author": {
            "name": "..."
            },
            "license": "MIT",
            "config": {
                "forge": {
                    "packagerConfig": {
                        // start the app with the start-app.sh script
                        "executableName": "start-app.sh",
                    },
                    "makers": [
                        {
                            "name": "@electron-forge/maker-squirrel",
                            "config": {
                                "name": "onedrive_app"
                                }
                                },
                                {
                                    "name": "@electron-forge/maker-zip",
                                    "platforms": [
                                        "darwin"
                                        ]
                                        },
                                        {
                                            "name": "@electron-forge/maker-deb",
                                            "config": {}
                                            },
                                            {
                                                "name": "@electron-forge/maker-rpm",
                                                "config": {}
                                                }
                                                ]
                                                }
                                                },
                                                "devDependencies": {
                                                    "@babel/core": "^7.4.5",
                                                    "@babel/preset-env": "^7.4.5",
                                                    "@babel/preset-react": "^7.0.0",
                                                    "@electron-forge/cli": "^6.0.0-beta.50",
                                                    "@electron-forge/maker-deb": "^6.0.0-beta.50",
                                                    "@electron-forge/maker-rpm": "^6.0.0-beta.50",
                                                    "@electron-forge/maker-squirrel": "^6.0.0-beta.50",
                                                    "@electron-forge/maker-zip": "^6.0.0-beta.50",
                                                    "@electron-forge/plugin-webpack": "^6.0.0-beta.50",
                                                    "babel-loader": "^8.0.6",
                                                    "css-loader": "^3.2.0",
                                                    "electron": "^6.0.0",
                                                    "electron-is-dev": "^1.1.0",
                                                    "eslint": "^6.1.0",
                                                    "eslint-config-airbnb": "^18.0.1",
                                                    "eslint-config-prettier": "^6.0.0",
                                                    "eslint-plugin-import": "^2.18.2",
                                                    "eslint-plugin-jsx-a11y": "^6.2.3",
                                                    "eslint-plugin-prettier": "^3.1.1",
                                                    "eslint-plugin-react": "^7.14.3",
                                                    "node-sass": "^4.12.0",
                                                    "prettier": "^1.18.2",
                                                    "react": "^16.8.6",
                                                    "react-dom": "^16.8.6",
                                                    "sass-loader": "^7.1.0",
                                                    "style-loader": "^1.0.0",
                                                    "webpack": "^4.35.2"
                                                    },
                                                    "dependencies": {
                                                        "electron-is-dev": "^1.1.0",
                                                        "electron-store": "^4.0.0",
                                                        "electron-updater": "^4.0.6",
                                                        "express": "^4.17.1",
                                                        "flask": "^1.1.1",
                                                        "flask-cors": "^3.0.8",
                                                        "flask-restful": "^0.3.7",
                                                        "flask-sqlalchemy": "^2.4.1",
                                                        "flask-wtf": "^0.14.3",
                                                        "python-shell": "^2.0.3",
                                                        "request": "^2.88.0",
                                                        "request-promise": "^4.2.4",
                                                        "sqlite3": "^4.1.1",
                                                        "wtf-forms": "^2.3.3"
                                                        }
                                                        }   
~~~
##### keep the app updated
<!-- To use an updater module in your Electron app, you'll need to include the module in your project and configure it to check for updates, download updates, and install them.

Here's an example using the electron-updater module:

Install the electron-updater module: 
npm install electron-updater
const { autoUpdater } = require('electron-updater');
autoUpdater.checkForUpdatesAndNotify();
autoUpdater.on('update-downloaded', (event, releaseNotes, releaseName) => {
  autoUpdater.quitAndInstall();
});

To host update files on GitHub, you can use GitHub Releases to distribute updates to your Electron app.

Here's an overview of how to use GitHub Releases to host update files:

Create a GitHub repository for your Electron app.

Create a new release in your repository by clicking on the "Releases" tab and then clicking the "Create a new release" button.

Upload the update files for your Electron app to the release.

In your Electron app, configure the updater module to download updates from the GitHub release. The exact configuration will depend on the updater module you're using, but typically you'll need to specify the URL to the update file and the type of update (e.g. zip, dmg, etc.).

When you want to release a new version of your Electron app, create a new release in GitHub with the updated files and the updater module will download and install the update for your users.

By using GitHub Releases to host updates, you can simplify the update process for your Electron app and provide a reliable and easy-to-use distribution method for your users. This approach can be especially useful if you have a small user base or if you don't need the advanced features of a more complex update server.
-->