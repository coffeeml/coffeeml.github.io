---
layout: post
comments: false
title:  "Building an MCP server and configuring with the Claude desktop app"
excerpt: "MCP is an open standard that promises to glue LLMs with tools and services, often called the USB-C of AI applications. LLMs frequently need to integrate with data and tools, and MCP standardizes these interactions."
date:   2025-07-04 10:00:00
---

[All about MCP](https://modelcontextprotocol.io/introduction). 

In this post I show how to create a simple MCP server and connect it with the Claude Desktop App (MCP client). [Download here](https://claude.ai/download).

Let's assume we are developing a conversation interface for a hotel booking app, where visitors can have a conversation with a chatbot and book a room. Let's also assume the chatbot is Claude Sonnet or any of Anthropic's LLMs, served via this Claude desktop app, which is our MCP client as well.

However, the LLM doesn't know anything about the hotel's internal processes and business information, like what rooms the hotel has, prices, amenities, etc. 

But what the hotel has, is its own enterprise software (like a reservation management system), which does all of that. Our job is to expose chosen functionalities as `tools` for the LLM, that the MCP client can help the LLM use, and facilitate the conversation with the visitors.

### ✅ Setup

Install `uv` as shown here: [UV documentation](https://docs.astral.sh/uv/getting-started/installation/).

### ✅ Create your workspace and open in vscode or any editor of your choice.

`cd` into the workspace and run:

```powershell
uv init HotelBookingMcp
uv venv .venv

.venv/Scripts/activate

uv add "mcp[cli]"

```

A simple `JSON` based in-memory data-structure in Python defining the hotel database as:

```python
# File: data/db.py

hotel_database = {
    "hotel_name": "Grand Hotel",
    "location": "123 Main St, Springfield, USA",
    "currency": "USD",
    "standard_amenities": ["breakfast", "free Wi-Fi"],
    "luxury_amenities": ["spa", "pool", "gym", "spa", ],
    "suite_amenities": ["private balcony", "ocean view"],
    "rooms": {
        "101": {
            "type": "standard",
            "price_per_night": 100.00,
            "available": True,
            "occupied_by": None,
        },
        "102": {
            "type": "standard",
            "price_per_night": 100.00,
            "available": True,
            "occupied_by": None,
        },
        "103": {
            "type": "standard",
            "price_per_night": 100.00,
            "available": True,
            "occupied_by": None,
        },
        "201": {
            "type": "luxury",
            "price_per_night": 150.00,
            "available": True,
            "occupied_by": None,
        },
        "202": {
            "type": "luxury",
            "price_per_night": 150.00,
            "available": True,
            "occupied_by": None,
        },
        "301": {
            "type": "suite",
            "price_per_night": 200.00,
            "available": True,
            "occupied_by": None,
        },
        "302": {
            "type": "suite",
            "price_per_night": 200.00,
            "available": True,
            "occupied_by": None,
        }        
    }
}
```

A collection of data access funtions defined as: 

```python
# File: utils/utils.py

from data.db import hotel_database as hdb

def get_hotel_info():
    """
    Returns the hotel name and location from the database.
    """
    return {
        "hotel_name": hdb["hotel_name"],
        "location": hdb["location"]
    }


def get_available_rooms():
    """
    Returns a list of available rooms in the hotel.
    """
    available_rooms = []
    for room_number, room_info in hdb["rooms"].items():
        if room_info["available"]:
            available_rooms.append({
                "room_number": room_number,
                "type": room_info["type"],
                "price_per_night": room_info["price_per_night"]
            })
    return available_rooms


def get_amenities():
    """
    Returns the standard, luxury, and suite amenities of the hotel.
    """
    return {
        "standard_amenities": hdb["standard_amenities"],
        "luxury_amenities": hdb["standard_amenities"] + hdb["luxury_amenities"],
        "suite_amenities": hdb["standard_amenities"] + hdb["luxury_amenities"] + hdb["suite_amenities"]
    }


def book_room_by_room_number(room_number, guest_name):
    """
    Books a room by room number for a guest if it is available.
    """
    if room_number in hdb["rooms"]:
        room_info = hdb["rooms"][room_number]
        if room_info["available"]:
            room_info["available"] = False
            room_info["occupied_by"] = guest_name
            return f"Room {room_number} booked successfully for {guest_name}."
        else:
            return f"Room {room_number} is already occupied."
    else:
        return f"Room {room_number} does not exist." 
    

def book_room_by_type(room_type, guest_name):
    """
    Books a room of a specific type for a guest if available.
    """
    for room_number, room_info in hdb["rooms"].items():
        if room_info["type"] == room_type and room_info["available"]:
            room_info["available"] = False
            room_info["occupied_by"] = guest_name
            return f"Room {room_number} of type {room_type} booked successfully for {guest_name}."
    return f"No available rooms of type {room_type} found."
    

def get_booking_details(room_number):
    """
    Returns the booking details of a specific room.
    """
    if room_number in hdb["rooms"]:
        room_info = hdb["rooms"][room_number]
        return {
            "room_number": room_number,
            "type": room_info["type"],
            "price_per_night": room_info["price_per_night"],
            "available": room_info["available"],
            "occupied_by": room_info["occupied_by"]
        }
    else:
        return f"Room {room_number} does not exist."
    

def count_available_rooms_by_type(room_type):
    """
    Counts the number of available rooms of a specific type.
    """
    count = 0
    for room_info in hdb["rooms"].values():
        if room_info["type"] == room_type and room_info["available"]:
            count += 1
    return count if count > 0 else f"No available rooms of type {room_type} found." 


def get_room_price_by_type(room_type):
    """
    Returns the price per night for a specific room type.
    """
    for room_number, room_info in hdb["rooms"].items():
        if room_info["type"] == room_type:
            return {
                "type": room_info["type"],
                "price_per_night": room_info["price_per_night"],
                "total_available": count_available_rooms_by_type(room_type)
            }
    return f"No rooms of type {room_type} found."

```

Data access util functions wrapped in MCP tools as:

```python
# File: tools/tools.py

from server import mcp
from utils import utils


@mcp.tool()
def get_hotel_info():
    """
    Fetches the hotel name and location.

    Returns:
        dict: A dictionary containing the hotel name and location.
    """
    return utils.get_hotel_info()


@mcp.tool()
def get_available_rooms():  
    """
    Fetches a list of available rooms in the hotel.

    Returns:
        list: A list of dictionaries, each containing room number, type, and price per night.
    """
    return utils.get_available_rooms()


@mcp.tool()     
def get_amenities():
    """
    Fetches the standard, luxury, and suite amenities of the hotel.

    Returns:
        dict: A dictionary containing lists of standard, luxury, and suite amenities.
    """
    return utils.get_amenities()  


@mcp.tool()
def book_room_by_room_number(room_number: str, guest_name: str):            
    """
    Books a room by its number for a guest if it is available.

    Args:
        room_number (str): The room number to book.
        guest_name (str): The name of the guest booking the room.

    Returns:
        str: A message indicating the result of the booking attempt.
    """
    return utils.book_room_by_room_number(room_number, guest_name)    


@mcp.tool()
def book_room_by_type(room_type: str, guest_name: str):
    """
    Books a room of a specific type for a guest if available.

    Args:
        room_type (str): The type of room to book (e.g., "standard", "luxury", "suite").
        guest_name (str): The name of the guest booking the room.

    Returns:
        str: A message indicating the result of the booking attempt.
    """
    return utils.book_room_by_type(room_type, guest_name)


@mcp.tool()
def get_booking_details(room_number: str):
    """
    Fetches the booking details for a specific room number.

    Args:
        room_number (str): The room number to get booking details for.

    Returns:
        dict: A dictionary containing booking details such as guest name, room type, and price per night.
    """
    return utils.get_booking_details(room_number)


@mcp.tool()
def count_available_rooms_by_type(room_type: str):      
    """
    Counts the number of available rooms of a specific type.

    Args:
        room_type (str): The type of room to count (e.g., "standard", "luxury", "suite").

    Returns:
        int: The number of available rooms of the specified type.
    """
    return utils.count_available_rooms_by_type(room_type) 


@mcp.tool()
def get_room_price_by_type(room_type: str): 
    """
    Fetches the price per night for a specific type of room.

    Args:
        room_type (str): The type of room to get the price for (e.g., "standard", "luxury", "suite").

    Returns:
        float: The price per night for the specified room type.
    """
    return utils.get_room_price_by_type(room_type)    

```

✅ Note the `@mcp.tool()` decorator. <br />
✅ Every tool function must begin with docstrings. This helps the LLM understand which tool does what! *Very important!*

Creating the server:

```python
# File: server.py

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Hotel Booking MCP Server")

```

Creating the entrypoint:

```python
# File: main.py

from server import mcp

# following import statements are necessary for the MCP server to recognize the tools
from tools.tools import (
    get_hotel_info,
    get_available_rooms,
    get_amenities,
    book_room_by_room_number,
    book_room_by_type,
    get_booking_details,
    count_available_rooms_by_type,
    get_room_price_by_type
)


if __name__ == "__main__":
    mcp.run(transport="stdio")
    
```

Your workspace should look like:

```text
workspace
  .venv
  data
    db.py
  utils
    utils.py
  tools
    tools.py
  server.py
  main.py

```

<hr />

The `mcp.run(transport="stdio")` in `main.py` indicates that the MCP server is running in the same host the client (in this case Claude Desktop App) is running.

## Claude desktop config to attach MCP server

Open Claude desktop app:

`File` ➡️ `Settings` ➡️ `Developer`

Edit config to edit the `claude_desktop_config.json` file.

On Windows, this file lives in `C:\Users\<username>\AppData\Roaming\Claude` directory.

⚠️ Copy-paste and modify one of the following configs in the `claude_desktop_config.json` file as required.

### ✅ Claude config for `stdio`

This is when the MCP server is running on the same machine as the MCP client.

Point `command` to the `python` in your `.venv`. On powershell you get the python path using command `(Get-Command python).Path`, assuming your `.venv` is active.
Point `args` to the absolute path of the `main.py` file.

```json
{
  "mcpServers": {
    "hotel-booking-mcp": {
      "command": "<absolute path>\\HotelBookingMcp\\.venv\\Scripts\\python.exe",
      "args": [
        "<absolute path>\\HotelBookingMcp\\main.py"
      ]
    }
  }
}
```

<hr />

### ⚠️ However, it's very unlikely that your MCP server and MCP client will run on the same host.

You will ideally have the MCP server running on a different host across the network.

Therefore, let's change `mcp.run(transport="stdio")` in `main.py` to `mcp.run(transport="streamable-http")`.

This indicates that your Claude Desktop App (MCP client) now needs to find the remote MCP server over HTTP, that is running somewhere accross the internet.

*Here the MCP server runs on 127.0.0.1, but if you have a remote server you can deploy it there and try with the remote IP address.*

After making the change in `main.py`, start the MCP server as:

```powershell
uv run .\main.py

INFO:     Started server process [15164]
INFO:     Waiting for application startup.
INFO      StreamableHTTP session manager started  
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
...

```

### ✅ Claude config for `streamable-http` to connect with remote MCP server

As explained, this is the configuration you need when the MCP server is running on a remote system across the network.

```json
{
  "mcpServers": {
    "hotel-booking-mcp": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "http://127.0.0.1:8000/mcp"
      ]
    }
  }
}
```

⚠️ After config changes, exit the Claude desktop app from `File` ➡️ `Exit` and start it again.

Either way, now when you ask the Sonnet LLM questions about the hotel, it will use one of the MCP tools available to it via the MCP client to find and provide you an answer.

Let's ask a question about the hotel, also note the list of MCP tools available to Claude desktop app now:

<img src="/assets/claude-mcp-1.jpg" width="70%"> <br />

Note for each answer, the app is showing which tool it used to figure out the answer, based on the docstrings we wrote earlier.

Then I'm asking about the list of available rooms:

<img src="/assets/claude-mcp-2.jpg" width="70%"> <br />

Now I'm asking about amenities and booking a suite room:

<img src="/assets/claude-mcp-3.jpg" width="70%"> <br /><br />
<img src="/assets/claude-mcp-4.jpg" width="70%"> <br /><br />
<img src="/assets/claude-mcp-5.jpg" width="70%"> <br /><br />

Next time when I ask for available rooms, the booked room 301 is excluded.

<img src="/assets/claude-mcp-6.jpg" width="70%"> <br />
