# Voice Assistant App Service

* Proposal: [SDL-0300](0300-Voice-Assistant-App-Service.md)
* Author: [Joey Grover](https://github.com/joeygrover)
* Status: **Returned for Revisions**
* Impacted Platforms: [ Core / HMI / RPC / iOS / Java Suite / JavaScript Suite ]


## Introduction

This proposal adds a new "Voice Assistant" app service for apps that provide voice assistant features. This is built on top of the App Services feature accepted in proposal SDL - 0167. This proposal is more simplified than previous examples but extends the feature for more than stand alone voice assistant applications.

## Motivation
We should provide app services for common app types that can integrate with the head unit or other SDL applications. Voice assistant apps are a big part of any user's life in the vehicle and could include dedicated voice assistants (Google, Amazon Alexa), or apps with local voice assistant capabilities (like Navigation apps that take destinations via voice). We should provide additional ways for these apps to integrate with the system.

## Proposed solution
### Voice Assistant App Service (MOBILE\_API / HMI\_API Changes)
```xml
<enum name="AppServiceType" platform="documentation">
    <!-- Existing values -->
    <element name="VOICE_ASSISTANT" since="X.X"/>
</enum>

	
<struct name="VoiceAssistantServiceManifest" since="X.X">

   <param name="wakeWords" type="String" array="true" minsize="1" mandatory="false"/>

   <param name="audioInputCapabilities" type="String" array="true" minsize="1" mandatory="false">
       <description>Values should be of enum type AudioInputCapabilities.</description>
   </param>
 
   <param name="supportsFailureFallback" type="boolean" mandatory="false">
       <description>A flag to indicate that this service will let the system know it has failed to parse the voice data into actionable outcomes.</description>
   </param>

    <param name="onlyActiveWhenVisible" type="boolean" mandatory="false">
       <description>A flag to indicate that this service should only be considered the active service when it is visible to the user.</description>
   </param>
</struct>
	
	
<struct name="VoiceAssistantServiceData" since="X.X">
    <description>This data is related to what a voice assistant service would provide.</description>

	<param name="unhandledParsedVoiceData" type="VoiceRecognitionResult" array="true" mandatory="false">
	   <description>This array will contain parsed voice data that the service was unable to handle. This will be helpful for a pass-back to a default voice assistant like an embedded service that will handle other SDL app's VR synonyms. </description>
	</param>
</struct>

<enum name="AudioInputMode" since="X.X">
    <element name="AUDIO_PASSTHRU"/>
    <element name="MICROPHONE_DIRECT"/>
</enum>

<enum name="VoiceAssistantTrigger" since="X.X">
    <element name="PUSH_TO_TALK_BUTTON"/>
    <element name="WAKE_WORD"/>
</enum>

<struct name="VoiceRecognitionResult" since="X.X">
	<param name="data" type="String" mandatory="true">
	</param>
	
	<param name="confidenceScore" type="Integer" minValue="0" maxValue="100" mandatory="true">
	</param>
</struct>
```

#### New Function `OnVoiceSessionStarted`

```xml
<function name="OnVoiceSessionStarted" functionID="OnVoiceSessionStartedID" messagetype="notification" since="X.X">
    <param name="triggerSource" type="VoiceAssistantTrigger" mandatory="true">
        <description>Informs the voice assistant of which source triggered the event.</description>
    </param>
    <param name="triggerInfo" type="String" mandatory="false">
        <description>Any extra information about the trigger. If the the user voiced a wake word and combined it with a command, this should be the fully recognized string including wake word and command, eg "Hey [App Name], navigate me home." The raw voice data should also be included as bulk data when available.</description>
    </param>
    <param name="audioInputMode" type="AudioInputMode" mandatory="true">
        <description>Informs the voice assistant of which audio input mode is assumed to be the one in use.</description>
    </param>
    <param name="containsData" type="Boolean" mandatory="true">
        <description>If true, the bulkData of this RPC contains initial microphone data that the user voiced along with your wake word.</description>
    </param>
    <param name="recognizedWakeWord" type="String" mandatory="false">
        <description>The wake word that was recognized, if one was recognized.</description>
    </param>
</function>
```

### Handled RPCs
- `OnVoiceSessionStarted` (new)

### Initiate a Voice Session with Active Service
If a voice assistant app service is active, when the user presses the "Push-To-Talk" button or equivalent, this should initiate a voice session based on the supplied `audioInputCapabilities` in the manifest.

1. If the service has the capability to obtain audio data directly from the microphone's source (phone's microphone, Bluetooth connected microphone, etc) this should be used as the preferred method. The application will be responsible for displaying an alert to the user in this use case.
2. If the service has listed the ability to parse audio data via `AUDIO_PASSTHRU`, the normal flow for the audio passthrough feature should be used.

### Pass Back of Failed Parsed Voice Data
If the voice assistant service fails to handle the recognized voice audio data as something within the context of the app or the parsed command is beyond its control, the data will be passed through an update to its `VoiceAssistantServiceData` in the form of an array of `VoiceRecognitionResult` items. This info can be passed to subscribers of this data which should include the onboard voice recognition/assistant that handles the SDL app VR synonyms. 

### Switching Voice Assistant Services
1. The original flow of app services and becoming the active app service applies with a new caveat. 

2. If the voice assistant service has the `onlyActiveWhenVisible` flag set to `true`, the voice assistant service will only be active when the user has put that app into `HMI_FULL`. This should not include `HMI_LIMITED` unless the reason it is in limited is because of an alert type of dialog or menu being displayed.

3. Once the app is no longer in an HMI level that allows for the service to be active, either the default or previously active voice assistant service with their `onlyActiveWhenVisible` set to `false` will be be reactivated.

## Potential downsides
1. Adding an ability to be a voice assistant service for apps like navigation complicates the overall flow of the service. However, by limiting this to only `HMI_FULL` status, we can create an expected flow that must be followed.

## Impact on existing code
This would have impact on the RPC spec, which impacts Core and all app libraries. Additionally, changes will need to be made to the Policy Server and Developer Portal for this new app service to add a new request from developers to use this app service.

## Alternatives considered
1. In the original proposal, a voice assistant app service was much more involved. It would actually be given VR synonyms provided by other SDL apps to build out VR grammars and actions. This version keeps things much simpler and allows more apps to be voice assistants. 
2. Classifying this service as a Voice Recognition service with a capability of being an assistant in the manifest. That way it becomes a little less cloudy of what the service does. The change would be easy to make if the SDLC decides that is a better option to go.
