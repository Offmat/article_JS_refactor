Open-closed principle - do I ever have to use it?
Most developers have heard about open-closed principle - one of Uncle Bob’s SOLID principles. It sounds reasonable, but it can still be a little bit blurry until first usage on ‘live’ code. Full state of principle is: ‘software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification’.

So what it really means?
We met development problem, which has shown us what open-closed principle really is for. In one of web applications we had form with two sections: demand channels and dynamic filters (used to filter dimensions). User can add as many filters as he wishes, but there were some channels which were disabling some of filters.
Demand channels: AD_EXCHANGE, HEADER_BIDDING, RESERVATION, OTHER
Dynamic filters(dimensions): website, ad_unit, geo, creative_size, device

First implementation of the problem was simple:

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

As you can see, website is supposed to be available only for AD_EXCHANGE channel. But last thing to say about code is that it’s permanent or static. So we have more requests from our client:

Let’s add another channel !

Diff for ResearchFormStateUpdater:
```js
   _updateDynamicFilters () {
     (this._getScopedSelector('.dynamic-filter')).each((_, filter) => {
-      (filter).trigger('dynamicFilter:disableWebsites', this.isWebsitesDimensionDisabled);
+     (filter).trigger('dynamicFilter:disableWebsites', this._areFormStateDimensionsDisabled());
     });
   }

_areFormStateDimensionsDisabled () {
  return this._shouldDisableFields(ResearchFormStateUpdater.DISABLING_DEMAND_CHANNELS);
}


-ResearchFormStateUpdater.WEBSITE_DISABLING_DEMAND_CHANNELS = [
+ResearchFormStateUpdater.DISABLING_DEMAND_CHANNELS = [
   '#research_form_demand_channels_hb',
   '#research_form_demand_channels_reservation',
-  '#research_form_demand_channels_other'
+  '#research_form_demand_channels_other',
+  '#research_form_demand_channels_ebda'

```

So now let’s add a new dimension - ‘product’ which is supposed to be available only for AD_EXCHANGE as well
Diff for ResearchFormStateUpdater:
```js
-      $(filter).trigger('dynamicFilter:disableWebsites', this.areFormStateDimensionsDisabled);
+      $(filter).trigger('dynamicFilter:disableWebsitesAndProducts', this.areFormStateDimensionsDisabled);
```

Diff for ResearchDynamicFilter:
```js
_setDynamicFilterDisableWebsitesEvent () {
-    $(this._getBody()).on('dynamicFilter:disableWebsites', (event, shouldDisableWebsites) => {
-      this._setFilterDisabledState(shouldDisableWebsites);
-      this._setMethodSelectWebsiteOptionDisabledState(shouldDisableWebsites);
+  _setDynamicFilterDisableWebsitesAndProductsEvent () {
+    $(this._getBody()).on('dynamicFilter:disableWebsitesAndProducts', (event, shouldDisableWebsitesAndProducts) => {
+      const selectedDimension = this._getDimension().find('option:selected').val();
+      if (selectedDimension === 'website') {
+        this._setWebsitesFilterDisabledState(shouldDisableWebsitesAndProducts);
+      } else if (selectedDimension === 'product') {
+        this._setProductsFilterDisabledState(shouldDisableWebsitesAndProducts);
+      }
+      this._setMethodSelectWebsiteAndProductOptionDisabledState(shouldDisableWebsitesAndProducts);
     });
   }
```

As we can see, with new PO request we had to change logic of main method responsible for switching filters availability. But it’s not over yet!

Let’s go bigger and add some switcher above channel. Let me introduce you to ‘Source’. It gets funny here.
Sources: Ad Manager, Ssp.
Demand channel available only for Ad Manager source.
‘Website’ is only dimension (so filter) available for Ssp source.

Diff for ResearchFormStateUpdater:
```js
+  _adManagerSourceCallbacks () {
+    (...)
+    this._updateDefaultStateOfDynamicFilters();
+    this._updateAdManagerDynamicFilters();
+  }

+  _sspSourceCallbacks () {
+    (...)
+    this._updateDefaultStateOfDynamicFilters();
+    this._updateSspDynamicFilters();
+  }

+  _updateDefaultStateOfDynamicFilters () {
+    $(this._getScopedSelector('.dynamic-filter')).each((_, filter) => {
+      $(filter).trigger('dynamicFilter:enableSspFilters', this.isSourceSsp);
+    });
+  }
+
+  _updateSspDynamicFilters () {
     $(this._getScopedSelector('.dynamic-filter')).each((_, filter) => {
-      $(filter).trigger('dynamicFilter:disableWebsitesAndProducts', this.areFormStateDimensionsDisabled);
+      $(filter).trigger('dynamicFilter:disableNonSspOptions', true);
     });
   }

+  _updateAdManagerDynamicFilters () {
+    $(this._getScopedSelector('.dynamic-filter')).each((_, filter) => {
+      $(filter).trigger('dynamicFilter:disableWebsitesAndProducts', this.areFormStateDimensionsDisabled && !this.isSourceSsp);
+    });
+  }
```

Diff for ResearchDynamicFilter:
```js






Now DynamicFilter has to know, which dimensions are for which source. We do have ResearchFormStateUpdater, why shouldn’t he be in charge?  
