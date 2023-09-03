# TouchOSC code for persisting data via MIDI Sysex
Stuffing data into tag properties of controls is a popular way of "persisting" data in TouchOSC panels. The tag data can be either JSON or a custom delimited format -- the important part is that it is an ASCII string representation of a given panel's data that needs to be stored across sessions or moved from one session to another or even shared over the Internet.

It would be very useful if the data stuffed into a given control's tag property could be emitted as a sysex message that could be stored to a file for backup or sharing purposes. This code snipped aims to solve that problem.

The example panel contains two functions: __sendSysexDump()__ and __handleSysexMessage()__

# sendSysexDump()
When this method is called, a control's data will be serialized into a sysex message and transmitted out the given connection ID.

The function takes three parameters:
* ctrlName - The name of the control to retrieve data from
* opcode - Can be 0 or 1.
   ```
   0 - Get data from the control's TAG property
   1 - Get data from the control's values.text property (works for LABEL and TEXT controls)
  ```
* connId - the ID (1 - 5) of the TouchOSC MIDI Connection to use when sending the sysex message

# handleSysexMessage()
This message parses a given sysex message and loads the data from the sysex payload back into the control. Typically, you would call this from the built-in onReceiveMIDI() callback.

The function will locate the specified control in the sysex payload, and then set the appropriate property based on the message's opcode. Robust error handling is implemented to reject malformed or invalid messages.

# Example Panel
This repo contains an example panel. Activating the "Dump to Sysex" button will send the data in the text control (with the Lord Varys quote) out via sysex message on connection ID 1. This sysex data can be captured by your favorite sysex librarian (such as MIDI-OX). If you play the sysex message back to TouchOSC (on any connection ID), it will reload the quote text back into the example panel's text control.

![Panel Screen Cap](https://github.com/MidiHax/touchosc-ctrl-sysex/blob/main/preview.png)

# Links
 * [Hexler TouchOSC](https://hexler.net/touchosc)
 * [TouchOSC Scripting API](https://hexler.net/pub/touchosc/scripting-api.html)
 * [MIDI-OX](http://www.midiox.com/)
 * [Roland Checksum algorithm](https://www.vguitarforums.com/smf/index.php?topic=20544.0)

# The Code
```
function sendSysexDump(ctrlName, opcode, connId)
  -- Get the data we want to serialize
  -- as a string - this could be the tag
  -- or values.text property. If the tag
  -- is storing JSON or delimited values
  -- then use the tag property.

  local payload = {}
  payload[1] = 0xF0 -- Start Sysex block
  payload[2] = 0x00 -- Sysex manufacturer ID
  payload[3] = 0x02
  payload[4] = 0x5A
  payload[5] = opcode -- 0 = Tag, 1 = values.text
  
  -- Write control name to sysex message
  for i=1,#ctrlName do
    table.insert(payload, string.byte(string.sub(ctrlName, i, i)))
  end
  
  table.insert(payload, 0x00) -- Null terminator 
  
  -- Get the data to include in the sysex message
  local control = root:findByName(ctrlName, true);
  local data
  if (opcode == 0) then
    data = control.tag
  else
    data = control.values.text
  end

  -- Write the control's data to the sysex message
  for i=1,#data do
    local b = string.byte(string.sub(data, i, i))
    if (b > 127) then
      print(ctrlName, ' contains invalid char')
      return
    end
    table.insert(payload, b)
  end
  
  table.insert(payload, 0x00) -- Null terminator 
  
  -- Compute the checksum (Roland algorithm)
  local checksum = 0
  for i=5, #payload do
    checksum = checksum + payload[i]
  end
  checksum = checksum % 128
  checksum = 128 - checksum
  
  table.insert(payload, checksum)
  table.insert(payload, 0xF7)
  
  -- Send sysex message
  local connections = {}
  for i=1,5 do
    if (connId == i) then
      connections[i] = true
    else
      connections[i] = false
    end
  end
  
  sendMIDI(payload, connections)
end

function handleSysexMessage(message)
  -- Verify this is a sysex message
  if (message[1] ~= MIDIMessageType.SYSTEMEXCLUSIVE) then
    return
  end
  
  -- Check ID
  if (message[2] ~= 0x00 or message[3] ~= 0x02 or message[4] ~= 0x5A) then
    return
  end
  
  -- Read the control name, starting at byte 6, until we hit null
  local idx = 6
  local ctrlName = ""
  while (message[idx] ~= 0x00) do
    ctrlName = ctrlName .. string.char(message[idx])
    idx = idx + 1
  end
  
  local targetCtrl = root:findByName(ctrlName, true)
  
  if (not targetCtrl) then
    print("Sysex dump references unknown control", ctrlName)
    return
  end
  
  -- Read the data, starting at the next byte after control name
  idx = idx + 1
  local data = ""
  while (message[idx] ~= 0x00) do
    data = data .. string.char(message[idx])
    idx = idx + 1
  end
  
  -- Set the control's property, based on opcode in 5th byte
  if (message[5] == 0x00) then
    targetCtrl.tag = data
  else
   targetCtrl.values.text = data
  end
end
```
