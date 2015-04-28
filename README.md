# Ember Lazy Load Images Cookbook

## Problem
When a website loads, below the fold images get requested alongside above the fold images.

## Solution
Write a scroll service that loads images ONLY when they are in viewport...

## Discussion
Here is where we discuss...

First we set up a sweet scroll service.
```javascript
import Ember from 'ember';

var debounceDuration = 30;
export default Ember.Object.extend(Ember.Evented, {
  scrollableElement: null,
  _eventQueue: (function() {
    return [];
  })(),

  _setupListener: function() {
    var el = this.get('scrollableElement');
    var $el = Ember.$(el);
    if ($el.length) {
      var service = this;
      Ember.$(window).on('scroll', function fallBackScrollHandler() {
        Ember.run.debounce(service, '_triggerEvents', debounceDuration);
      });
      $el.on('scroll', function scrollHandler() {
        Ember.run.debounce(service, '_triggerEvents', debounceDuration);
      });
    }
  }.observes('scrollableElement'),

  _triggerEvents: function() {
    var eventQueue = this.get('_eventQueue');
    if (!Ember.isEmpty(eventQueue)) {
      // iterate through the queue and trigger all the events...
      for (var i = 0, len = eventQueue.length; i < len; i++) {
        this.trigger(eventQueue[i]);
      }
    }
  },

  subscribe: function(element, eventName) {
    var currentElement = this.get('scrollableElement');
    // check to see if the scrollableElement is set
    if (currentElement == null) {
      this.set('scrollableElement', element);
    } else if (currentElement !== element) {
      // TODO: handle more than one scroller?
      // TODO: flush queue and remove scroll handler?
      console.log(
        'Scroll service is already watching ',
        currentElement,
        ' and is not currently set up to watch more than one!'
      );
      return;
    }
    this.get('_eventQueue').pushObject(eventName);
  },

  unsubscribe: function(element, eventName) {
    var eventQueue;
    var currentElement = this.get('scrollableElement');
    if (currentElement === element) {
      // remove event from queue
      eventQueue = this.get('_eventQueue');
      eventQueue.removeObject(eventName);
    } else if (currentElement !== element) {
      // TODO: handle more than one scroller?
      // TODO: flush queue and remove scroll handler?
      return;
    }

    // if queue empty, remove listener and null out scrollableElement
    if (Ember.isEmpty(eventQueue)) {
      Ember.$(currentElement).off('scroll');
      Ember.$(window).off('scroll');
      this.set('scrollableElement', null);
    }
  }
});

```

Then, make it easy to subscribe to with SubscribeToScroll Mixin...
```javascript
import Ember from 'ember';

export default Ember.Mixin.create({
  _scrollElement: '.js-page',

  scrollDepth: Ember.inject.service(),

  _scrollHandler: function() {
    console.warn('`_scrollHandler` not implemented!');
  },

  _subscribeToScroll: function() {
    var _self = this;
    var scrollEvent = this.get('_scrollEvent');
    if (Ember.isBlank(scrollEvent)) {
      console.warn('`_scrollEvent` not implemented!');
      return;
    }
    var scrollElement = this.get('_scrollElement');
    Ember.run.schedule('afterRender', function() {
      var scrollDepthService = _self.get('scrollDepth');
      scrollDepthService.on(scrollEvent, _self, '_scrollHandler');
      scrollDepthService.subscribe(scrollElement, scrollEvent);
    });
  }.on('didInsertElement'),

  _unsubscribeToScroll: function() {
    var scrollDepthService = this.get('scrollDepth');
    var scrollEvent = this.get('_scrollEvent');
    var scrollElement = this.get('_scrollElement');
    scrollDepthService.unsubscribe(scrollElement, scrollEvent);
    scrollDepthService.off(scrollEvent);
  }.on('willDestroyElement'),

});

```

