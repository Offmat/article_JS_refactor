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

Spoiler alert -> when component is open for changes, there will be a lot of name changing in future. We won't pay attention to this in next steps.

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

### Implementation with new functionality:
I will leave full callbacks list here, so you can see it wasn't only about dynamic filters - there was a lot of functionality.

```js
class ResearchFormStateUpdater {
  update () {
    (...)
    this._triggerCallbacks();
  }

  _triggerCallbacks () {
    if (this.isSourceSsp) {
      this._sspSourceCallbacks();
    } else {
      this._adManagerSourceCallbacks();
    }
  }

  _adManagerSourceCallbacks () {
    this._toggleDimensions(ResearchFormStateUpdater.SSP_DIMENSIONS, true);
    this._enableDemandChannels(ResearchFormStateUpdater.AD_MANAGER_DEMAND_CHANNELS);
    this._toggleDimensions(ResearchFormStateUpdater.AD_MANAGER_DIMENSIONS, false);
    this._updateFormStateUpdaterAttributes();
    this._updateDemandChannelsLabelClasses();
    this._updateDimensions();
    this._updateDefaultStateOfDynamicFilters();
    this._updateAdManagerDynamicFilters();
  }

  _sspSourceCallbacks () {
    this._removeDemandChannelsActiveClassAndDisable(ResearchFormStateUpdater.AD_MANAGER_DEMAND_CHANNELS);
    this._toggleDimensions(ResearchFormStateUpdater.AD_MANAGER_DIMENSIONS, true);
    this._enableDemandChannels(ResearchFormStateUpdater.SSP_DEMAND_CHANNELS);
    this._toggleDimensions(ResearchFormStateUpdater.SSP_DIMENSIONS, false);
    this._updateDefaultStateOfDynamicFilters();
  }

  _updateDefaultStateOfDynamicFilters () {
    $(this._getScopedSelector('.dynamic-filter')).each((_, filter) => {
      $(filter).trigger('dynamicFilter:enableSspFilters', this.isSourceSsp);
    });
  }

  _updateAdManagerDynamicFilters () {
    $(this._getScopedSelector('.dynamic-filter')).each((_, filter) => {
      $(filter).trigger('dynamicFilter:disableWebsitesAndProducts', this._areFormStateDimensionsDisabled() && !this.isSourceSsp);
    });
  }

  _areFormStateDimensionsDisabled () {
    this._shouldDisableFields(ResearchFormStateUpdater.AD_MANAGER_DISABLING_DEMAND_CHANNELS)
  }

  _shouldDisableFields (disablingDemandChannels) {
    return disablingDemandChannels.find(selector => {
      return $(this._getScopedSelector(selector)).prop('checked');
    }) !== undefined;
  }
}

ResearchFormStateUpdater.AD_MANAGER_DISABLING_DEMAND_CHANNELS = [
  '#research_form_demand_channels_hb',
  '#research_form_demand_channels_reservation',
  '#research_form_demand_channels_other',
  '#research_form_demand_channels_ebda'
];

class ResearchDynamicFilter {
  _setDefaultDynamicFiltersToggleEvent () {
    $(this._getBody()).on('dynamicFilter:enableSspFilters', (event, shouldEnableSspOptions) => {
      this._setDefaultFiltersOptionDisabledState(shouldEnableSspOptions);

      const selectedFilterDimension = this._getFiltersDimension().find('option:selected').val();
      if (selectedFilterDimension === 'website') {
        this._toggleChosenFilterDisabledState(false);
      } else if (selectedFilterDimension === 'platform') {
        this._toggleChosenFilterDisabledState(!shouldEnableSspOptions);
      } else {
        this._toggleChosenFilterDisabledState(shouldEnableSspOptions);
      }
    });
  }

  _setDynamicFilterDisableWebsitesAndProductsEvent () {
    $(this._getBody()).on('dynamicFilter:disableWebsitesAndProducts', (event, shouldDisableWebsitesAndProducts) => {
      const selectedFilterDimension = this._getFiltersDimension().find('option:selected').val();
      if ($.inArray(selectedFilterDimension, ['website', 'product']) >= 0) {
        this._toggleChosenFilterDisabledState(shouldDisableWebsitesAndProducts);
      }
      this._setMethodSelectWebsiteAndProductOptionDisabledState(shouldDisableWebsitesAndProducts);
    });
  }

  _toggleNonSspFilters (dimensionSelect, shouldDisable) {
    $.each(ResearchDynamicFilter.NON_SSP_FILTERS_OPTIONS, (_, option) => {
      dimensionSelect.find(option).prop('disabled', shouldDisable);
    });
  }
}

ResearchDynamicFilter.NON_SSP_FILTERS_OPTIONS = [
  'option[value="ad_unit"]', 'option[value="creative_size"]',
  'option[value="geo"]', 'option[value="device"]',
  'option[value="product"]'
];
```
We still use some *'toggle'* mechanism. It is really hard to switch 5 levers and get to expected state and now DynamicFilter has to know, which dimensions are not for ssp source. We do have ResearchFormStateUpdater, why shouldn’t he be in charge?

## Final request
So now lets add additional dimension - Yield partner.
