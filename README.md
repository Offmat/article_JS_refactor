# Open-closed principle - do I ever have to use it?
Most developers have heard about open-closed principle - one of Uncle Bob’s SOLID principles. It sounds reasonable, but it can still be a little bit blurry until first usage on ‘live’ code. Full state of principle is: *software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification*.

## So what does it really mean?
We came across development problem, which has shown us what open-closed principle really is about. In one of our web applications we had form with two sections (among others):
* demand channels
* dynamic filters  
User can add as many filters as he wishes, but there are some rules - filter availability depends on chosen channels.  
Demand channels: AD_EXCHANGE, HEADER_BIDDING, RESERVATION, OTHER  
Dynamic filters(dimensions): website, ad_unit, geo, creative_size, device  

This article is mostly about code refactor, so there will be a lot of code snippets below. I tried to reduce it, but some amount of code is necessary to show code refactoring. You don't need to understand every small part of code to get the main idea.

### First implementation of the problem was simple:

```js
class ResearchFormStateUpdater {
  update () {
    (...)
    this._updateDynamicFilters();
  }

  _updateDynamicFilters () {
    $('.dynamic-filter').each((_, filter) => {
      $(filter).trigger('dynamicFilter:disableWebsites', this._shouldDisableWebsitesFields());
    });
  }

  _shouldDisableWebsitesFields () {
    return this._shouldDisableFields(ResearchFormStateUpdater.WEBSITE_DISABLING_DEMAND_CHANNELS);
  }

  _shouldDisableFields (disablingDemandChannels) {
    // is any of disablingDemandChannels checked ?
  }
}

ResearchFormStateUpdater.WEBSITE_DISABLING_DEMAND_CHANNELS = ['header_bidding', 'reservation', 'other'];

class ResearchDynamicFilter {
  _setDynamicFilterDisableWebsitesEvent () {
    $(this._getBody()).on('dynamicFilter:disableWebsites', (event, shouldDisableWebsites) => {
      // disable website filters
    });
  }
}
```

As you can see, website filter is supposed to be available only for AD_EXCHANGE channel (unavailable for rest channels).  
Last thing you can say about code is that it is permanent or static. So we have more requests from our client making these classes bigger and more complex.  

## Feature development
* **Add another channel - EBDA** (Website filter should be unavailable while EBDA is chosen):
  * expand *DISABLING_DEMAND_CHANNELS* by ebda demand channel
  * a lot of name changing - in first implementation we specified website in methods and constants names. For example:
    * *isWebsitesDimensionDisabled* to *_areFormStateDimensionsDisabled*
    * *WEBSITE_DISABLING_DEMAND_CHANNELS* to *DISABLING_DEMAND_CHANNELS*

Spoiler alert -> when component is open for changes, there will be a lot of name changing in future. We won't pay attention to this in next steps.

* **Add another filter for 'Product'** (Product filter availability scheme is same as Website)
  * *ResearchDynamicFilter* class has to check for one more dimension while disabling/enabling fields

* **Let’s go bigger and add some switcher above channels -> ‘Source’:**
  * Rules:
    * There are two states od source: Ad Manager, Ssp.
    * All of our demand channels are available only for Ad Manager source.
    * There are no demand channels for Ssp source
    * ‘Website’ is only filter available for Ssp source.
  * Implementation:
    * When 'Ssp' is chosen:
      * Disable demand channels.
      * trigger *'dynamicFilter:disableWebsitesAndProducts'* <- enable both
      * trigger *'dynamicFilter:disableNonSspOptions'*
    * When Ad Manager checked:
      * trigger *'dynamicFilter:disableWebsitesAndProducts'* <- check weather enable or disable

* **Add another dfilter for 'Platform'**
  * Rules:
    * Platform is available only when source is Ssp
  * Difficulty:
    * Now we have website, which is available for AD_EXCHANGE channel from Ad Manager and for Ssp and we have product which is available for Ssp but not for Ad Manager
    * *Toggling* state of form gets really tricky and confusing

### Implementation with new functionality:
I present you this snippet mainly to show code complexity. Feel free to flip through it.

```js
class ResearchFormStateUpdater {
  update () {
    (...)
    this._triggerCallbacks();
  }

  _triggerCallbacks () {
    // choose callbacks depending on source
  }

  _adManagerSourceCallbacks () {
    (...)
    this._enableDemandChannels(ResearchFormStateUpdater.AD_MANAGER_DEMAND_CHANNELS);
    this._updateDefaultStateOfDynamicFilters();
    this._updateAdManagerDynamicFilters();
  }

  _sspSourceCallbacks () {
    (...)
    this._removeDemandChannelsActiveClassAndDisable(ResearchFormStateUpdater.AD_MANAGER_DEMAND_CHANNELS);
    this._updateDefaultStateOfDynamicFilters();
  }

  _updateDefaultStateOfDynamicFilters () {
    $('.dynamic-filter').each((_, filter) => {
      $(filter).trigger('dynamicFilter:enableSspFilters', this.isSourceSsp);
    });
  }

  _updateAdManagerDynamicFilters () {
    $('.dynamic-filter').each((_, filter) => {
      $(filter).trigger('dynamicFilter:disableWebsitesAndProducts', this._areFormStateDimensionsDisabled() && !this.isSourceSsp);
    });
  }

  _areFormStateDimensionsDisabled () {
    this._shouldDisableFields(ResearchFormStateUpdater.AD_MANAGER_DISABLING_DEMAND_CHANNELS)
  }

  _shouldDisableFields (disablingDemandChannels) {
    // is any of disablingDemandChannels is checked
  }
}

ResearchFormStateUpdater.AD_MANAGER_DISABLING_DEMAND_CHANNELS = ['header_bidding', 'reservation', 'other', 'ebda'];

class ResearchDynamicFilter {
  // I didn't simplify those two methods body to show current implementation complexity

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
      // toggle filter state depending on 'shouldDisable'
    });
  }
}

ResearchDynamicFilter.NON_SSP_FILTERS_OPTIONS = ['ad_unit', 'creative_size', 'geo', 'device', 'product'];
```

We still use some *'toggle'* mechanism. It is really hard to switch 4 levers and get to expected state and now DynamicFilter has to know, which dimensions are not for ssp source. We do have ResearchFormStateUpdater, why shouldn’t he be in charge?

## Final request
### **Add another filter for 'Yield partner'**  
That is exact moment when we decided to refactor those classes. Channels and filters we are analysing are just small part of problem. We have a lot of form sections here and all off them have same problem. Our refactor should neutralise the need of changing inside methods of those classes in order to add some new channels or dimensions.  
In next snippet I left main classes almost as they are in our production code to show you how easy to understand they are now.

```js
class ResearchFormStateUpdater {
  update () {
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
    $('.dynamic-filter').each((_, filter) => {
      this._toggleDynamicFilterState(filter, disabledFilters);
    });
  }

  _toggleDynamicFilterState (dynamicFilter, disabledFilters) {
    $(dynamicFilter).trigger('dynamicFilter:toggleDynamicFilters', disabledFilters);
  }
}

ResearchFormStateUpdater.NO_SSP_FILTERS = ['ad_unit', 'creative_size', 'geo', 'device', 'product'];

ResearchFormStateUpdater.ONLY_SSP_FILTERS = ['platform'];

ResearchFormStateUpdater.ONLY_ADX_FILTERS = ['website', 'product'];

class ResearchDynamicFilter {
  _setDynamicFiltersToggleEvent () {
    $(this._getBody()).on('dynamicFilter:toggleDynamicFilters', (event, disabledFilters) => {
      this._disableFilters(disabledFilters.split(','));
      this._enableFilters(disabledFilters.split(',')));
    });
  }

  _disableFilters (filtersToDisable) {
    // disable filtersToDisable
  }

  _enableFilters (filtersToDisable) {
    const filtersToEnable = $(ResearchDynamicFilter.ALL_FILTERS).not(filtersToDisable).get();
    // enable filtersToEnable
  }
}

ResearchDynamicFilter.ALL_FILTERS = ['website', 'ad_unit', 'creative_size', 'geo', 'device', 'product', 'platform'];
```

## We did it! Did we?
Now the only thing 'ResearchDynamicFilter' has to know is list of all filters - seems fair. Rest of logic and control comes from above - some higher methods and constants.  
So lets try out our new structure with adding filter for 'Yield_partner':

```js
class ResearchFormStateUpdater {
  _dynamicFiltersDimensionsToBeDisabled () {
    (...)
    if (this.areDemandChannelsExceptEbdaSelected) {
      disabledFilters = disabledFilters.concat(ResearchFormStateUpdater.ONLY_EBDA_FILTERS);
    }
    return disabledFilters;
  }
}

ResearchFormStateUpdater.NO_SSP_FILTERS = [(...), 'yield_partner'];

ResearchFormStateUpdater.ONLY_EBDA_FILTERS = [(...), 'yield_partner'];

ResearchDynamicFilter.ALL_FILTERS = [(...), 'yield_partner'];
```

As you can see, it is all about adding some values to constants and some additional condition.  

Thanks to *'open-closed principle'* we are able to change business logic of form with only adding some values and conditions on higher level of abstraction. We don't need to go inside component and change anything. This refactor affected whole form and there were more sections and they all obey open-closed principle now.  
We didn't reduce amount of code - as a matter of fact we even increased it (before/after):
* *ResearchFormStateUpdater* - 211/282 lines
* *ResearchDynamicFilter* - 267/256 lines  

It's all about collection in constants -> its our public interface now, our console to control process without tens of switchers.
