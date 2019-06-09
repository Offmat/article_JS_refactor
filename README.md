# Open-closed principle - do I ever have to use it?
Most developers have heard about open-closed principle - one of Uncle Bob’s SOLID principles. It sounds reasonable, but it can still be a little bit blurry until first usage on ‘live’ code. Full state of principle is: *software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification*.

## So what it really means?
We met development problem, which has shown us what open-closed principle really is about. In one of web applications we had form with two sections (among others): demand channels and dynamic filters. User can add as many filters as he wishes, but there were some channels which were disabling some of filters.  
Demand channels: AD_EXCHANGE, HEADER_BIDDING, RESERVATION, OTHER  
Dynamic filters(dimensions): website, ad_unit, geo, creative_size, device  

### First implementation of the problem was simple:

```js

class ResearchFormStateUpdater {
  update () {
    (...)
    this._updateDimensions();
  }

  _updateDynamicFilters () {
    $(this._getScopedSelector('.dynamic-filter')).each((_, filter) => {
      $(filter).trigger('dynamicFilter:disableWebsites', this._shouldDisableWebsitesFields());
    });
  }

  _shouldDisableWebsitesFields () {
    return this._shouldDisableFields(ResearchFormStateUpdater.WEBSITE_DISABLING_DEMAND_CHANNELS);
  }

  _shouldDisableFields (disablingDemandChannels) {
    return disablingDemandChannels.find(selector => {
      return $(this._getScopedSelector(selector)).prop('checked');
    }) !== undefined;
  }
}

ResearchFormStateUpdater.WEBSITE_DISABLING_DEMAND_CHANNELS = [
  '#research_form_demand_channels_hb',
  '#research_form_demand_channels_reservation',
  '#research_form_demand_channels_other'
];

class ResearchDynamicFilter {
  _setDynamicFilterDisableWebsitesEvent () {
    $(this._getBody()).on('dynamicFilter:disableWebsites', (event, shouldDisableWebsites) => {
      this._setFilterDisabledState(shouldDisableWebsites);
      this._setMethodSelectWebsiteOptionDisabledState(shouldDisableWebsites);
    });
  }
}
```

As you can see, website is supposed to be available only for AD_EXCHANGE channel. But last thing you can say about code is that it is permanent or static. So we have more requests from our client, and making those classes bigger and more complex.

* **Adding another channel - EBDA:**
  * expand *'ResearchFormStateUpdater.DISABLING_DEMAND_CHANNELS'* by ebda demand channel
  * name changing:
    * *'isWebsitesDimensionDisabled'* to *'_areFormStateDimensionsDisabled'*
    * *'WEBSITE_DISABLING_DEMAND_CHANNELS'* to *'DISABLING_DEMAND_CHANNELS'*

Spoiler alert -> when component i open for changes, there will be a lot of name changing in future. We won't pay attention to this in next steps.

* **Adding another dimension - product:**
  * *'ResearchDynamicFilter'* has to check for one more dimension while disabling/enabling fields

* **Let’s go bigger and add some switcher above channels -> ‘Source’:**
  * Rules:
    * Sources: Ad Manager, Ssp.
    * Demand channels are available only for Ad Manager source.
    * ‘Website’ is only dimension (also filter) available for Ssp source.
  * Implementation:
    * When 'Ssp' checked:
      * Disable demand channels.
      * trigger *'dynamicFilter:disableWebsitesAndProducts'* <- enable both
      * trigger *'dynamicFilter:disableNonSspOptions'*
    * When Ad Manager checked:
      * trigger *'dynamicFilter:disableWebsitesAndProducts'* <- check weather enable or disable








Now DynamicFilter has to know, which dimensions are for which source. We do have ResearchFormStateUpdater, why shouldn’t he be in charge?  
