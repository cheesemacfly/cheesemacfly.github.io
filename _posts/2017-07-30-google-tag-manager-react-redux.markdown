---
title: Google Tag Manager and React/Redux
date: 2017-07-30 00:00:00 Z
categories:
- blog
tags:
- redux
- react
- js
- google
- tag
- manager
- analytics
layout: post
---

The Google Tag Manager dataLayer is one of the nicest thing when it comes to analytics for the developers. But there's one major issue when you start having 
custom dimensions: you need to reset the dataLayer between your pages or you will be sending the same data for a specific propery over and over again.

Let's take for example the following dataLayer:

{% highlight javascript %}
dataLayer.push({
  userType = 'customer',
  productType = 'food',
  foodName = 'burger'
});
{% endhighlight %}

And now your user goes to the next page that is not a product of type food anymore:

{% highlight javascript %}
dataLayer.push({
  userType = 'customer',
  productType = 'drink'
});
{% endhighlight %}

If your page hasn't been reloaded between the 2 `push()`, the following data will be sent to Google Tag Manager:

{% highlight javascript %}
{
  userType = 'customer',
  productType = 'drink',
  foodName = 'burger'
};
{% endhighlight %}

This will not be an issue if your website isn't a single page app (each page reload 
will flush the dataLayer since it gets recreated) but it can be a huge problem if 
you build your website using react and redux which are commonly used for single page applications.

I'll show you how to create and use a middleware that solves this issue using a hidden method of the GTM library.

First we have to find a way to reset the dataLayer. The Google documentation 
is not helpful here...so I reached out to Simo Ahava on Twitter to get the answer:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">google_tag_manager[“GTM-XXXX”].dataLayer.reset() will clear GTM’s internal data model.</p>&mdash; Simo Ahava (@SimoAhava) <a href="https://twitter.com/SimoAhava/status/823753362843770888">January 24, 2017</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

We are going to create a very simple reducer to handle the data that we want to push to the dataLayer.  
It will handle 3 cases: 

  - adding data to the state (ADD_TO_DATALAYER)
  - adding data and pushing it to the gtm dataLayer (PUSH_TO_DATALAYER)
  - reseting the gtm dataLayer (RESET_DATALAYER)

Here's the reducer contained in one file:

{% highlight javascript %}
export const Types = {
  ADD_TO_DATALAYER: 'app/analytics/ADD_TO_DATALAYER',
  PUSH_TO_DATALAYER: 'app/analytics/PUSH_TO_DATALAYER',
  RESET_DATALAYER: 'app/analytics/RESET_DATALAYER'
};

const DEFAULT_STATE = {
  dataLayer: null
};

export default function reducer(state = DEFAULT_STATE, action = {}) {
  switch (action.type) {
    case Types.ADD_TO_DATALAYER:
      return {
        ...state,
        dataLayer: {
          ...state.dataLayer,
          ...action.data
        }
      };
    case Types.RESET_DATALAYER:
      return {
        ...state,
        dataLayer: null
      };
    default:
      return state;
  }
}

export const addToDataLayer = (data = {}) => {
  return {
    type: Types.ADD_TO_DATALAYER,
    data
  };
};

export const pushToDataLayer = (data = {}) => (dispatch) => {
  dispatch(addToDataLayer(data));
  return dispatch({
    type: Types.PUSH_TO_DATALAYER
  });
};

export const resetDataLayer = () => {
  return {
    type: Types.RESET_DATALAYER
  };
};

export const getDataLayer = (state) => {
  return state.dataLayer;
};
{% endhighlight %}

Now that we have a way to reset the dataLayer and our reducer we can move on to our next step: creating a middleware.

{% highlight javascript %}
import { Types, getDataLayer } from 'app/redux/analytics'; //assuming this is the path to the reducer we just created

// the name of the container for GTM, usually 'dataLayer'
const DATA_LAYER_CONTAINER = 'dataLayer';

const _dataLayer = window[DATA_LAYER_CONTAINER] || [];
const _gtm = window.google_tag_manager['GTM-XXXX']; //replace with your container Id

export default ({ getState }) => {
  return (next) => (action) => {
    const returnValue = next(action);

    if (action.type === Types.PUSH_TO_DATALAYER) {
      _dataLayer.push(getDataLayer(getState()));
    } else if (action.type === Types.RESET_DATALAYER) {
      _gtm[DATA_LAYER_CONTAINER].reset();
    }

    return returnValue;
  };
};
{% endhighlight %}

The last step will be to plug in the middelware:

{% highlight javascript %}
//somewhere in the file that creates your store
const store = applyMiddleware(analyticsMiddleware/*, thunk, routerMiddleware...*/)(createStore)(myReducer);
{% endhighlight %}

This is not a complete solution and using an undocumented feature from Google Tag Manager can be risky but it does work.
Also note that you can simplify the reducer even more if you only use action creators and types (skipping the state).  
I'm also not including the interaction with React but it should be pretty straight forward following the doc: <http://redux.js.org/docs/basics/UsageWithReact.html>