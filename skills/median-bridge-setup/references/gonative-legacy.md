# GoNative / Median Legacy Bridge Reference

For apps last updated on the GoNative.io platform, the global object and
protocol are `gonative` rather than `median`. Median-built apps accept both.

Source: https://docs.median.co/page/gonative-javascript-bridge

*Ported verbatim from the upstream GoNative bridge documentation; the source page above no longer resolves (404), so these mappings cannot be re-verified against live docs. Two entries look like upstream-docs quirks — preserved as-is but unverified: (1) `navigationTitles.revert` posts to `gonative://navigationTitles/set?persist=true`, a `set` endpoint with a query flag rather than any dedicated `revert` endpoint; (2) `navigationLevels.set` and `navigationLevels.setCurrent` both target the identical `gonative://navigationLevels/set` URL, so the two functions cannot behave differently. Verify both against a live GoNative-era app build before relying on them.*

```javascript
///////////////////////////////
////    General Commands   //// 
///////////////////////////////

gonative.nativebridge = {
    custom: function (params){
        addCommand("gonative://nativebridge/custom", params);
    }
};

gonative.registration = {
    send: function(params){
        addCommand("gonative://registration/send", params);
    }
};

gonative.sidebar = {
    setItems: function (params){
        addCommand("gonative://sidebar/setItems", params);
    }
};

gonative.tabNavigation = {
    selectTab: function (tabIndex){
        addCommand('gonative://tabs/select/' + tabIndex);
    },
    setTabs: function (params){
        addCommand('gonative://tabs/setTabs', params);
    }
};

gonative.share = {
    sharePage: function (params){
        addCommand("gonative://share/sharePage", params);
    },
    downloadFile: function (params){
        addCommand("gonative://share/downloadFile", params);
    }
};

gonative.open = {
    appSettings: function (){
        addCommand("gonative://open/app-settings");
    }
};

gonative.webview = {
    clearCache: function(){
        addCommand("gonative://webview/clearCache");
    }
};

gonative.config = {
    set: function(params){
        addCommand("gonative://config/set", params);
    }
};

gonative.navigationTitles = {
    set: function (params){
        addCommand("gonative://navigationTitles/set", params);
    },
    setCurrent: function (params){
        addCommand("gonative://navigationTitles/setCurrent", params);
    },
    revert: function(){
        addCommand("gonative://navigationTitles/set?persist=true");
    }
};

gonative.navigationLevels = {
    set: function (params){
        addCommand("gonative://navigationLevels/set", params);
    },
    setCurrent: function(params){
        addCommand("gonative://navigationLevels/set", params);
    },
    revert: function(){
        addCommand("gonative://navigationLevels/set?persist=true");
    }
};

gonative.statusbar = {
    set: function (params){
        addCommand("gonative://statusbar/set", params);
    }
};

gonative.screen = {
    setBrightness: function(params){
        addCommand("gonative://screen/setBrightness", params);
    }
};

gonative.navigationMaxWindows = {
    set: function (params){
        addCommand("gonative://navigationMaxWindows/set", params);
    }
};

gonative.connectivity = {
    get: function (params){
        return addCommandCallback("gonative://connectivity/get", params);
    },
    subscribe: function (params){
        return addCommandCallback("gonative://connectivity/subscribe", params);
    },
    unsubscribe: function (){
        addCommand("gonative://connectivity/unsubscribe");
    }
};

gonative.run = {
    deviceInfo: function(){
        addCommand("gonative://run/gonative_device_info");
    },
    onesignalInfo: function(){
        addCommand("gonative://run/gonative_onesignal_info");
    }
};

// onesignal
gonative.onesignal = {
    register: function (){
        addCommand("gonative://onesignal/register");
    },
    userPrivacyConsent:{
        grant: function (){
            addCommand("gonative://onesignal/userPrivacyConsent/grant");
        },
        revoke: function (){
            addCommand("gonative://onesignal/userPrivacyConsent/revoke");
        }
    },
    tags: {
        getTags: function(params){
            return addCommandCallback("gonative://onesignal/tags/get", params);
        },
        setTags: function (params){
            addCommand("gonative://onesignal/tags/set", params);
        }
    },
    showTagsUI: function () {
        addCommand("gonative://onesignal/showTagsUI");
    },
    promptLocation: function () {
        addCommand("gonative://onesignal/promptLocation");
    },
    iam: {
        addTrigger: function (params){
            addCommand("gonative://onesignal/iam/addTrigger", params);
        },
        addTriggers: function (params){
            addCommand("gonative://onesignal/iam/addTriggers", params);
        },
        removeTriggerForKey: function (params){
            addCommand("gonative://onesignal/iam/removeTriggerForKey", params);
        },
        getTriggerValueForKey: function (params){
            addCommand("gonative://onesignal/iam/getTriggerValueForKey", params);
        },
        pauseInAppMessages: function (){
            addCommand("gonative://onesignal/iam/pauseInAppMessages?pause=true");
        },
        resumeInAppMessages: function (){
            addCommand("gonative://onesignal/iam/pauseInAppMessages?pause=false");
        },
        setInAppMessageClickHandler: function (params){
            addCommand("gonative://onesignal/iam/setInAppMessageClickHandler", params);
        }
    }
};

// facebook
gonative.facebook = {
    events: {
        send: function(params){
            addCommand("gonative://facebook/events/send", params);
        },
        sendPurchase: function(params){
            addCommand("gonative://facebook/events/sendPurchase", params);
        }
    }
};

///////////////////////////////
////     iOS Exclusive     ////
///////////////////////////////

gonative.ios = {};

gonative.ios.window = {
    open: function (params){
        addCommand("gonative://window/open", params);
    }
};

gonative.ios.geoLocation = {
    requestLocation: function () {
        addCommand("gonative://geolocationShim/requestLocation");
    },
    startWatchingLocation: function () {
        addCommand("gonative://geolocationShim/startWatchingLocation");
    },
    stopWatchingLocation: function () {
        addCommand("gonative://geolocationShim/stopWatchingLocation");
    }
};

gonative.ios.attconsent = {
    request: function (params){
        return addCommandCallback("gonative://ios/attconsent/request", params);
    },
    status: function (params){
        return addCommandCallback("gonative://ios/attconsent/status", params);
    }
};

gonative.ios.backgroundAudio = {
    start: function(){
        addCommand("gonative://backgroundAudio/start");
    },
    end: function(){
        addCommand("gonative://backgroundAudio/end");
    }
};

///////////////////////////////
////   Android Exclusive   ////
///////////////////////////////

gonative.android = {};

gonative.android.geoLocation = {
    promptAndroidLocationServices: function(){
        addCommand("gonative://geoLocation/promptAndroidLocationServices");
    }
};

gonative.android.screen = {
    fullscreen: function(){
        addCommand("gonative://screen/fullscreen");
    },
    normal: function(){
        addCommand("gonative://screen/normal");
    },
    keepScreenOn: function(){
        addCommand("gonative://screen/keepScreenOn");
    },
    keepScreenNormal: function(){
        addCommand("gonative://screen/keepScreenNormal");
    }
};
```
