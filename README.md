
# Suggesting Places with jQuery, Geolocation, and HERE Maps, part 2


## By Chris Hamaide | 27 November 2018


A few weeks ago,  Nic Raboy had published [an article](https://developer.here.com/blog/suggesting-places-with-jquery-geolocation-and-the-here-places-api) describing how to suggest places around a location using **HERE Maps Places**, combined with **jQuery**.

In this tutorial, we will see how one can also make use of **HERE Geocoder Autocomplete REST API**, and even combine both Places and Geocoder for best results.


While Nic was using Handlebars.js to compile HTML and Typeahead.js for the typeahead logic, we’ll use this time [jQuery UI](https://jqueryui.com/autocomplete), which is a natural fit with jQuery.

To do so, you’ll have to include in the header section of your HTML file the following link:
 
    <link rel="stylesheet" href="https://code.jquery.com/ui/1.12.1/themes/base/jquery-ui.css">

And at the end of the body section this library:

     <script src="https://code.jquery.com/ui/1.12.1/jquery-ui.js"></script>

While we’re at it, let’s also put our javascript code in a separate file, index.js, for sake of clarity:

     <script src="index.js"></script>


Our index.html is therefore this:

    <html>
    <head>
         <title>HERE Auto-complete</title>
          <link rel="stylesheet" href="https://code.jquery.com/ui/1.12.1/themes/base/jquery-ui.css">
    </head>
    <body>
          <div>
          <p id="coords">Location: searching....</p>
          <input size="40" id="place" type="text" placeholder="Search a Place...">
          <input size="40" id="address" type="text" placeholder="Search an Address...">
          <input size="40" id="full" type="text" placeholder="Search Something...">
         </div>

          <script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
          <script src="https://code.jquery.com/ui/1.12.1/jquery-ui.js"></script>
          <script src="index.js"></script>

    </body>
    </html>


## Hmm, but wait !
Why 3 input fields, identified by **place**, **address** and **full** ?

Well, in order to compare the results from **Place Autosuggest**, **HERE Geocoder autocomplete** and the **combination of the two**, this exemple will allow you to compare the results of each option. Nice, no ?

Let’s go now to the javascript part, with a bit of help from jQuery.

Remember that in order to provide the suggestion from Places autosuggest API, the source of typeahead autocomplete was the following:

     source: function (query, callback) {
          $.getJSON("https://places.cit.api.here.com/places/v1/autosuggest?at=" + coordinates + "&q=" + query + "&app_id=APP_ID_HERE&app_code=APP_CODE_HERE", function (data) {
               var places = data.results.filter(place => place.vicinity);
               places = places.map(place => {
               place.title = place.title.replace(/<br\/>/g, ", ");
               place.vicinity = place.vicinity.replace(/<br\/>/g, ", ");
               return place;
          });
          return callback(places);
     });


With jQuery Autocomplete, the mechanism is very much similar, only difference being, that you have to provide the following information:

*  **title**:  what is displayed in the list of suggestions
*   **value**: what is displayed in the input field when you have selected a suggestion

So let’s transform above code into this:
     
     source: function (query, callback) {
          $.getJSON("https://places.cit.api.here.com/places/v1/autosuggest?at=" + coordinates + "&q=" + query.term + "&app_id=" + APP_ID_HERE + "&app_code=" + APP_CODE_HERE, function (data) {
               let places = data.results.filter(place => place.vicinity);
               places = places.map(place => {
               return {
                    title: place.title,
                    value: place.title + ', ' + place.vicinity.replace(/<br\/>/g, ", ") + ' (' + place.category + ')',
                    id: place.id
               };
          });
          return callback(places);
     });


And we’re good to go with jQuery Autocomplete.


## Wait, wait wait ! 
Why replacing **return place**  by **return {title:…, value:…, id:…}** ?

Well, Place contains a lot of various lengthy fields, so let’s ease the task of jquery UI...


## Wait, wait wait ! 
What is this **id** field ? 

This field bears a unique identifier, which one can later use as an input for the HERE geocoder to get the latitude, longitude of the selection. But that will be for another tutorial..

We now bind the first input field to the jQuery autocomplete as such:

     $("#place").autocomplete({
          source: placeAC,
          minLength: 3,
          select: function (event, ui) {
               console.log("Selected: " + ui.item.value + " with LocationId " + ui.item.id);
          }
     });

And the function placeAC is what we’ve seen above:

     function placeAC(query, callback) {
          $.getJSON("https://places.cit.api.here.com/places/v1/autosuggest?at=" + coordinates + "&q=" + query.term + "&app_id=" + APP_ID_HERE + "&app_code=" + APP_CODE_HERE, function (data) {
               let places = data.results.filter(place => place.vicinity);
               places = places.map(place => {
               return {
                    title: place.title,
                    value: place.title + ', ' + place.vicinity.replace(/<br\/>/g, ", ") + ' (' + place.category + ')',
                    id: place.id
               };
          });
          return callback(places);
     });
     

Note that once a suggestion is selected, the function is called:

    select: function (event, ui) {
        console.log("Selected: " + ui.item.value + " with LocationId " + ui.item.id);
    }

Here, we just display the value and the unique Location id. One would ultimately use this to trigger the geocoding of the suggestion.

## Let's now bring the Geocoder autocomplete

Now let’s consider the HERE geocoder autocomplete. It is very efficient at finding full addresses, and works extra fast. You can find its full Developer’s guide on developer.here.com.

Usage is extremely similar to the Places autosuggest API :
* the query term is to be put into **query** parameter, 
* the center of the search into **prox**, 
* the array of results are provided into the field **suggestion**
  
Here you are:

     function addressAC(query, callback) {
          $.getJSON("https://autocomplete.geocoder.api.here.com/6.2/suggest.json?prox=" + coordinates + "&query=" + query.term + "&app_id=" + APP_ID_HERE + "&app_code=" + APP_CODE_HERE, function (data) {
               let addresses = data.suggestions;
               addresses = addresses.map(addr => {
               return {
                    title: addr.label,
                    value: addr.label,
                    id: addr.locationId
               };
          });
          return callback(addresses);
     });
     

And to bind the autocomplete with this source function to the second input field:

     $("#address").autocomplete({
          source: addressAC,
          minLength: 2,
          select: function (event, ui) {
               console.log("Selected: " + ui.item.value + " with LocationId " + ui.item.id);
          }
     });

So far so good ?  
Great, let’s combine the two together.

What we want to achieve is combine the result of two asynchronous function \$.getJSON, one fetching data from HERE Places autosuggest API, the other one from HERE Geocoder autocomplete API.
jQuery provides a convenient function for this : **$.when**

     let p1 = $.getJSON(….)
     let p2 = $.getJSON(…)
     $.when(p1, p2).done(function (data1, data2) {
          // data1[0] contains the result of the first $.getJSON, 
          // data2[0] contains the result of the second $.getJSON
     });

So, let’s build the arrays as before, one called **places**, managing the result of Places autosuggest, the other one called **addresses** with the result of the Geocoder autocomplete API:

     let places = data1[0].results.filter(place => place.vicinity);
     places = places.map(place => {
          return {
               title: place.title,
               value: place.title + ',' + place.vicinity.replace(/<br\/>/g, ", ") + '(' + place.category + ')',
               distance: place.distance,
               id: place.id
          };
     });

     // data2 is from address autocomplete
     let addresses = data2[0].suggestions;
     addresses = addresses.map(addr => {
          return {
               title: addr.label,
               value: addr.label + ' (address)',
               distance: addr.distance,
               id: addr.locationId
          };
     });

Now, let’s merge the two arrays into one:

        $.merge(places, addresses);

Results goes into the first array, **places**.

 let's sort the array by distance. Notice that both Places and Geocoder provides you with the distance of each suggestion to the center of the query. 
 Convenient no ?

        places.sort(function (p1, p2) { return p1.distance - p2.distance });

 and in order to limit the number of suggestions to a decent number (would you say 10 ?), let’s return a slice of the array:

        return callback(places.slice(0, 10));


Once you have  bound the third input field to the jQuery autocomplete,  you’re good to grab a cup of coffee and admire your work !

## Congratulation, you did it !


You may find the two files index.html and index.js on [Github](https://github.com/devbab/here-autocomplete-jquery) for you to download experiment.

## Happy coding !

[Chris Hamaide](mailto:christophe.hamaide@here.com)

*PS : Nic, you owe me one…*























