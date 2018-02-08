---
title: "Twitter Typeahead"
date: 2015-08-29T09:00:45-05:00
draft: false
---


Twitter's [Typeahead widget](https://twitter.github.io/typeahead.js/) adds suggestions to an input 
field based on what the user has already typed.  We're all familiar with such widgets from 
Google searches, for example.

![Typeahead example](/images/typeahead_example.png)

I thought it would be easy to set up and get going.  Indeed, seeing first (promising) results 
went really quickly.  But then I spent the rest of the day figuring out the corner cases that
weren't handled and Dr. Google didn't provide answers for.

**Update:** At some point I grew frustrated with Typeahead again and switched to 
[jQuery-Autocomplete](https://github.com/devbridge/jQuery-Autocomplete).

The Typeahead use case seems to be one input field in a larger form.  It helps find the correct
value for that field but something else in the form handles the submission.

My use case is a different but very common one -- search.  The form has a single search field
used to search for, in my case, a person.  The search could be on one of the unique identifiers
(a number or alphameric userid) or based on a name.  The name is where the typeahead is needed.
Once the user has found the right person (perhaps by selecting off the typeahead list), hit
enter and it's submitted.

The hard part was getting it to play nicely with the keyboard:  Submit on enter, submit the correct
suggestion if chosen with the arrow keys, avoid resubmitting contents the are already in the form
from a previous submission.

Finally, when the submission occurs it should be with the userid for uniqueness.  Each user's
userid is returned from my suggestion web service along with the name.

I think the basic issue is Typeahead's event model and API is incomplete.  I kept looking
in the documentation for two things:

1. A method to get the current suggestion
2. An event that says "Here's the suggestion that was chosen"

I found the `typeahead:change` event to be useless.  I could not figure out how it gets 
triggered.

The `typeahead:select` and `typeahead:autocomplete` were useful for when the user hit
tab or clicked in another field.  But not when they hit the enter key.

My suggestions web service returns json lists such as:

``` javascript
[
  {  "surname": "Bailey", "givennames": "Bea", "userid": "bbailey" },
  {  "surname": "Barker", "givennames": "Bob", "userid": "babarker" }
]
```

The HTML is

``` html
<div><form action="/search" method="GET" id="search">
	<input type="text" id="searchTerm" name="searchTerm" value="" 
		class="search form-control typeahead" autocomplete="off">
	</form></div>
```

Here's the Javascript I finally came up with:

``` javascript
asisJS = function () {


  var initSearchTypeahead = function () {

    /*
     * A hack to keep track of the currently selected userid
     * when the cursor keys are moved.
     */
    var currentQuery = "";

    /*
     * Cursor moved down the list of suggestions.  Keep track
     * of the current suggestion in currentQuery
     */
    function logCursorChange(ev, suggestion) {
      if (typeof suggestion == 'undefined' ) {
        currentQuery = "";
      } else {
        currentQuery = suggestion.userid;
      }
    }

    /*
     * User triggered a submit, either via the typeahead:submit 
     * or autocomplete event
     */
    function submitSuggestion(ev, suggestion) {
      $('#searchTerm').val(suggestion.userid);
      $('#search').submit();    // submit the form
    }

    /*
     * Initialize the search field with the typeahead widget
     * The source function is hacked so it gets hits if the first
     * character is in [A-Z] (a name).  Otherwise (eg uwid, userid, etc)
     * it does not.
     */
    $('#searchTerm')
      .typeahead(
      {
        minLength: 1
      }, {
        // Where to get the typeahead suggestions
        source: function (query, processSync, processAsync) {
          // If the field has contents (eg "who") that will be stored
          // in currentQuery before the user starts to type.  Don't want that.
          currentQuery = "";

          if (query.charAt(0) >= "A" && query.charAt(0) <= "Z") {
            // Get the autocomplete;  We don't have any synchronous
            // suggestions.  Call the suggestion API to get 10 suggestions
            // given the contents of the query.  The query may contain
            // spaces, letters, and numbers -- nothing else.  Replace
            // the spaces with _ for http transport.
            processSync([]);
            $.get('/api/v2/student/' + query.replace(/ /g, '_') + '/typeahead/10', 
            	function (json) {  processAsync(json.data);  }
            );
          } else {
            // Not a name; don't do the autocomplete
            processSync([]);
            processAsync([]);
            currentQuery = query;
          }
        },

        // How to display a suggestion
        display: function (suggestion) {
          return suggestion.surname + ", " + suggestion.givennames;
        },

        // Bug in Typeahead.
        // See https://github.com/twitter/typeahead.js/issues/1232
        limit: 20     

      })

       //Catch typeahead events
      .on('typeahead:select', submitSuggestion)
      .on('typeahead:autocomplete', submitSuggestion)
      .on('typeahead:cursorchange', logCursorChange)

      // Submit the form if the user hits "enter"
      .on('keydown', function (event) {
        if (event.which === 13) {
          if (currentQuery === '') {
            // Trigger the default (first) suggestion
            $('.tt-suggestion:first-child').trigger('click');
          } else {
            // The suggestion they chose with arrow keys
            $('#searchTerm').val(currentQuery);
            $('#search').submit();    // submit the form
          }
        }
      })
    ;

    // Select any text already in the textbox (eg from error handling)
    // when the page loads.
    $('#searchTerm').focus().select();
  };


  return {
    initSearchTypeahead: initSearchTypeahead
  };
}
();
```

There you have it;  hope it's been helpful!
