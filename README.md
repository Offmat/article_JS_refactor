# Open-closed principle - do I ever have to use it?
Most developers have heard about open-closed principle - one of Uncle Bob’s SOLID principles. It sounds reasonable, but it can still be a little bit blurry until first usage on ‘live’ code. Full state of principle is: *software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification*.

## So what it really means?
We met development problem, which has shown us what open-closed principle really is about. In one of our web applications we had form with two sections (among others): demand channels and dynamic filters. User can add as many filters as he wishes, but there are some rules - filter availability depends on chosen channels.  
Demand channels: AD_EXCHANGE, HEADER_BIDDING, RESERVATION, OTHER  
Dynamic filters(dimensions): website, ad_unit, geo, creative_size, device  

### First implementation of the problem was simple:

```js

class ResearchFormStateUpdater {
  update () {
    (...)
    this._updateDynamicFilters();
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

As you can see, website is supposed to be available only for AD_EXCHANGE channel. But last thing you can say about code is that it is permanent or static. So we have more requests from our client making these classes bigger and more complex.

* **Add another channel - EBDA:**
  * expand *'ResearchFormStateUpdater.DISABLING_DEMAND_CHANNELS'* by ebda demand channel
  * name changing:
    * *'isWebsitesDimensionDisabled'* to *'_areFormStateDimensionsDisabled'*
    * *'WEBSITE_DISABLING_DEMAND_CHANNELS'* to *'DISABLING_DEMAND_CHANNELS'*

Spoiler alert -> when component is open for changes, there will be a lot of name changing in future. We won't pay attention to this in next steps.

* **Add another dimension - product:**
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
* **Add ssp dimension - product**
  * Rules:
    * Product is available only when source is Ssp
  * Difficulty:
    * Now we have website, which is available for AD_EXCHANGE channel from Ad Manager and for Ssp and we have product which is available for Ssp but not for Ad Manager
    * *Toggling* state of form gets really tricky and confusing

### Implementation with new functionality:
I will leave full callbacks list here, so you can see it wasn't only about dynamic filters - there was a lot of sections with own functionality.

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
We still use some *'toggle'* mechanism. It is really hard to switch 4 levers and get to expected state and now DynamicFilter has to know, which dimensions are not for ssp source. We do have ResearchFormStateUpdater, why shouldn’t he be in charge?

## Final request
So now lets add additional dimension - Yield partner. That is exact moment when we decided to refactor.
```js
class ResearchFormStateUpdater {
  update () {
    (...)
    this._triggerCallbacks();
  }

  _triggerCallbacks () {
    (...)
    this._updateDynamicFilters();
  }

  _updateDynamicFilters () {
    this._toggleAllDynamicFiltersState(this._dynamicFiltersDimensionsToBeDisabled());
  }

  _dynamicFiltersDimensionsToBeDisabled () {
    if (this.isSourceSsp) { return ResearchFormStateUpdater.NO_SSP_FILTERS; }

    var disabledFilters = ResearchFormStateUpdater.ONLY_SSP_FILTERS;
    if (this.areDemandChannelsExceptAdxSelected) {
      disabledFilters = disabledFilters.concat(ResearchFormStateUpdater.ONLY_ADX_FILTERS);
    }
    return disabledFilters;
  }

  _toggleAllDynamicFiltersState (disabledFilters) {
    const serializedDisableFilters = disabledFilters.join(',');

    $(this._getScopedSelector('.dynamic-filter')).each((_, filter) => {
      this._toggleDynamicFilterState(filter, serializedDisableFilters);
    });
  }

  _toggleDynamicFilterState (dynamicFilter, disabledFilters) {
    $(dynamicFilter).trigger('dynamicFilter:toggleDynamicFilters', disabledFilters);
  }
}

ResearchFormStateUpdater.NO_SSP_FILTERS = [
  'ad_unit',
  'creative_size',
  'geo',
  'device',
  'product'
];

ResearchFormStateUpdater.ONLY_SSP_FILTERS = [
  'platform'
];

ResearchFormStateUpdater.ONLY_ADX_FILTERS = [
  'website',
  'product'
];

class ResearchDynamicFilter {
  _setDynamicFiltersToggleEvent () {
    $(this._getBody()).on('dynamicFilter:toggleDynamicFilters', (event, disabledFilters) => {
      this._toggleSelectedFilters(disabledFilters.split(','));
      this._toggleUnselectedFilters(this._formatedDimensionValues(disabledFilters.split(',')));
    });
  }

  _toggleUnselectedFilters (filtersToDisable) {
    const filtersToEnable = $(ResearchDynamicFilter.ALL_FILTERS).not(filtersToDisable).get();
    (...)
  }
}

ResearchDynamicFilter.ALL_FILTERS = [
  'option[value="website"]',
  'option[value="ad_unit"]',
  'option[value="creative_size"]',
  'option[value="geo"]',
  'option[value="device"]',
  'option[value="product"]',
  'option[value="platform"]'
];
```
And now to add new dimension there is no need to change anythong deep inside the code and bottom level methods. Lets just add some values to classes constants and maybe some additional condition:
```js
_dynamicFiltersDimensionsToBeDisabled () {
  (...)
  if (this.areDemandChannelsExceptEbdaSelected) {
    disabledFilters = disabledFilters.concat(ResearchFormStateUpdater.ONLY_EBDA_FILTERS);
  }
  return disabledFilters;
}

ResearchFormStateUpdater.NO_SSP_FILTERS = [
  (...)
  'yield_partner'
];

ResearchFormStateUpdater.ONLY_EBDA_FILTERS = [
  'yield_partner'
];

ResearchDynamicFilter.ALL_FILTERS = [
  (...)
  'option[value="yield_partner"]'
];
```
Thanks to *'open-closed principle'* we are able to change business logic of form with only adding some values and conditions on higher level of abstraction. We don't need to go inside component and change anything. This refactor affected whole form, so you can imagine there were some additional sections and it all works based on arrays of values. We didn't reduce amount of code - as a matter of fact we even increased it (before/after):
* *ResearchFormStateUpdater* - 211/282 lines
* *ResearchDynamicFilter* - 267/256 lines  
It's all about arrays of values in updater class - 83 lines now, but its our public interface, our console to control process without tens of switchers.
