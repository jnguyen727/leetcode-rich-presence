# LeetCode Discord Rich Presence

An extension and server application that updates your Discord Rich Presence status based on the LeetCode problem you're currently viewing.

---

## Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [1. Clone the Repository](#1-clone-the-repository)
  - [2. Set Up a Discord Application](#2-set-up-a-discord-application)
  - [3. Set Up the Python Server](#3-set-up-the-python-server)
  - [4. Set Up the Chrome Extension](#4-set-up-the-chrome-extension)
- [Configuration](#configuration)
- [Usage](#usage)
- [Troubleshooting](#troubleshooting)
- [Security and Privacy](#security-and-privacy)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

---

## Introduction

LeetCode Discord Rich Presence is a personal project that enhances your Discord profile by displaying the LeetCode problem you're currently working on. The integration automatically detects the problem open in your browser and updates your Discord Rich Presence with the problem title and difficulty.

---

## Features

- **Automatic Detection**: Detects the LeetCode problem you're viewing without manual input.
- **Real-Time Updates**: Updates your Discord status in real-time as you navigate between problems.
- **Problem Details**: Displays the problem title and difficulty level.
- **Clickable Link**: Includes a button to view the problem directly on LeetCode.
- **Easy Setup**: Simple installation and configuration process.

---

## Prerequisites

- **Discord Desktop Application**: Must be running on your computer.
- **Google Chrome Browser**: The extension is designed for Chrome (Manifest Version 3).
- **Python 3.6+**: For running the local server application.
- **pip**: Python package installer.
- **Git**: For cloning the repository.

---

## Installation

### 1. Clone the Repository

Open your terminal and run:

```bash
git clone https://github.com/yourusername/leetcode-discord-rich-presence.git
cd leetcode-discord-rich-presence
```

### 2. Set Up a Discord Application

1. **Visit the Discord Developer Portal**:
   - Go to [Discord Developer Portal](https://discord.com/developers/applications).
   - Log in with your Discord account.

2. **Create a New Application**:
   - Click **"New Application"**.
   - Enter a name (e.g., **"LeetCode Rich Presence"**) and click **"Create"**.

3. **Configure the Application**:
   - **General Information**:
     - Note down the **"Application ID"** (Client ID).
   - **Rich Presence Assets**:
     - Navigate to **"Rich Presence"** > **"Art Assets"**.
     - Upload images you plan to use (e.g., LeetCode logo).

### 3. Set Up the Python Server

#### a. Create a Virtual Environment (Optional but Recommended)

```bash
python -m venv venv
```

- **Activate the Virtual Environment**:
  - **Windows**:
    ```bash
    venv\Scripts\activate
    ```
  - **macOS/Linux**:
    ```bash
    source venv/bin/activate
    ```

#### b. Install Required Packages

```bash
pip install -r requirements.txt
```

- **If `requirements.txt` is not provided, install manually**:

  ```bash
  pip install flask flask_cors pypresence
  ```

#### c. Configure Environment Variables

- **Create a `.env` File**:

  Create a file named `.env` in the root directory:

  ```
  DISCORD_CLIENT_ID=your_discord_client_id_here
  ```

- **Ensure `.env` is in `.gitignore`**:

  ```gitignore
  # .gitignore
  .env
  venv/
  ```

### 4. Set Up the Chrome Extension

#### a. Update `manifest.json`

Ensure `manifest.json` is configured for Manifest Version 3:

```json
{
  "manifest_version": 3,
  "name": "LeetCode Discord Rich Presence",
  "version": "1.0",
  "description": "Updates Discord Rich Presence based on the LeetCode problem you're viewing.",
  "permissions": ["activeTab", "scripting"],
  "host_permissions": ["https://leetcode.com/problems/*", "http://localhost/*"],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["https://leetcode.com/problems/*"],
      "js": ["content_script.js"]
    }
  ]
}
```

#### b. Modify Extension Scripts

- **`background.js`**:

  Ensure the service worker handles messages correctly:

  ```javascript
  // background.js

  chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
    if (message.action === 'updatePresence') {
      fetch('http://localhost:5000/update_presence', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          title: message.title,
          difficulty: message.difficulty
        })
      })
        .then(response => response.text())
        .then(data => {
          console.log('Presence updated:', data);
          sendResponse({ status: 'success', data: data });
        })
        .catch(error => {
          console.error('Error updating presence:', error);
          sendResponse({ status: 'error', error: error.toString() });
        });

      // Keep the messaging channel open for sendResponse
      return true;
    }
  });
  ```

- **`content_script.js`**:

  Update the selectors if necessary:

  ```javascript
  // content_script.js

  window.addEventListener('load', () => {
    const problemTitleElement = document.querySelector('div[data-cy="question-title"]');
    const difficultyElement = document.querySelector('div[diff]');

    if (problemTitleElement && difficultyElement) {
      const problemTitle = problemTitleElement.textContent.trim();
      const problemDifficulty = difficultyElement.textContent.trim();

      chrome.runtime.sendMessage(
        {
          action: 'updatePresence',
          title: problemTitle,
          difficulty: problemDifficulty
        },
        (response) => {
          if (response && response.status === 'success') {
            console.log('Presence updated successfully.');
          } else if (response && response.status === 'error') {
            console.error('Error updating presence:', response.error);
          } else {
            console.warn('No response from background script.');
          }
        }
      );
    } else {
      console.error('Problem title or difficulty element not found.');
    }
  });
  ```

---

## Configuration

### Flask Server (`server.py`)

Ensure `server.py` reads the Discord Client ID from the environment:

```python
import os
from flask import Flask, request
from flask_cors import CORS
from pypresence import Presence
import time
import threading
import re

app = Flask(__name__)
CORS(app)

CLIENT_ID = os.getenv('DISCORD_CLIENT_ID')
RPC = Presence(CLIENT_ID)

def connect_rpc():
    try:
        RPC.connect()
        print("Connected to Discord RPC.")
    except Exception as e:
        print(f"Failed to connect to Discord RPC: {e}")

threading.Thread(target=connect_rpc).start()

@app.route('/update_presence', methods=['POST'])
def update_presence():
    data = request.json
    problem_title = data.get('title')
    problem_difficulty = data.get('difficulty')

    if problem_title and problem_difficulty:
        try:
            slug = re.sub(r"[^\w\s-]", '', problem_title.lower()).replace(' ', '-')
            RPC.update(
                state=f"Difficulty: {problem_difficulty}",
                details=f"Solving: {problem_title}",
                large_image='leetcode_logo',
                large_text='LeetCode',
                buttons=[{
                    "label": "View Problem",
                    "url": f"https://leetcode.com/problems/{slug}/"
                }],
                start=time.time()
            )
            print(f"Updated presence for problem: {problem_title}")
            return 'Discord Rich Presence updated successfully!', 200
        except Exception as e:
            print(f"Error updating Discord Rich Presence: {e}")
            return 'Failed to update Discord Rich Presence.', 500
    else:
        print("Invalid data received.")
        return 'Invalid data received.', 400

if __name__ == '__main__':
    app.run(port=5000)
```

---

## Usage

### 1. Run the Flask Server

Ensure your virtual environment is activated (if using one):

```bash
python server.py
```

You should see:

```
* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
Connected to Discord RPC.
```

### 2. Load the Extension in Chrome

- Open Chrome and navigate to `chrome://extensions/`.
- Enable **Developer mode** (toggle in the top-right corner).
- Click **"Load unpacked"** and select the extension folder.

### 3. Browse LeetCode Problems

- Go to [LeetCode](https://leetcode.com).
- Navigate to any problem page.
- Your Discord Rich Presence should update automatically.

---

## Troubleshooting

### Discord Rich Presence Not Updating

- **Ensure Discord is Running**: The desktop app must be open.
- **Check the Server**: Make sure `server.py` is running without errors.
- **Verify Client ID**: Ensure the `DISCORD_CLIENT_ID` in your `.env` file is correct.
- **Check for Errors**: Look at the console output of the server and Chrome DevTools.

### Extension Not Working

- **Extension Enabled**: Confirm the extension is loaded and enabled in `chrome://extensions/`.
- **Permissions**: Ensure the extension has the necessary permissions.
- **Check Console Logs**: Use Chrome DevTools to check for errors in the content script and background service worker.

### CORS Errors

- **Install `flask_cors`**: If not installed, run `pip install flask_cors`.
- **Enable CORS in `server.py`**: Ensure `CORS(app)` is called after initializing the Flask app.

### Selectors Not Matching

- **Update Selectors**: LeetCode may change their website structure. Inspect the page and update the selectors in `content_script.js` accordingly.

---

## Security and Privacy

- **Personal Use Only**: This project is intended for personal and educational purposes.
- **Respect Terms of Service**:
  - **LeetCode**: Ensure compliance with LeetCode's [Terms of Service](https://leetcode.com/terms/).
  - **Discord**: Adhere to Discord's [Developer Terms of Service](https://discord.com/developers/docs/policies-and-agreements/terms-of-service).
- **Protect Sensitive Data**:
  - Do not share your `DISCORD_CLIENT_ID` publicly.
  - Ensure your `.env` file is included in `.gitignore`.

---

## Contributing

Contributions are welcome! Please follow these guidelines:

- **Fork the Repository**: Create your own fork of the project.
- **Create a Feature Branch**: `git checkout -b feature/YourFeature`
- **Commit Your Changes**: `git commit -m "Add YourFeature"`
- **Push to the Branch**: `git push origin feature/YourFeature`
- **Open a Pull Request**: Describe your changes and submit.

---

## License

This project is licensed under the [MIT License](LICENSE).

---

## Acknowledgments

- **[Discord Developer Portal](https://discord.com/developers/applications)**
- **[LeetCode](https://leetcode.com)**
- **[pypresence](https://github.com/qwertyquerty/pypresence)**
- **[Flask](https://flask.palletsprojects.com/)**
- **[Flask-CORS](https://flask-cors.readthedocs.io/en/latest/)**
- **[Chrome Extensions Documentation](https://developer.chrome.com/docs/extensions/mv3/)**

---

**Note**: This project is for personal use and should not be distributed without ensuring compliance with all relevant terms and policies.

**Happy coding!**

---

## Contact

If you have any questions or need assistance, feel free to open an issue or reach out at [your-email@example.com](mailto:your-email@example.com).
