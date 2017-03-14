This document describes about the changes you need to be done when running DJANGO haystack with SOLR 6+.
Thanks for @github/nazariyg most of the configuration I learnt from following link
https://github.com/nazariyg/Solr-5-for-django-haystack

## Assumptions
-SOLR core is created and you know the path of the SOLR core

# Steps

### Configuring the solrconfig.xml

- Open **[SOLR_CORE_PATH]/conf/solrconfig.xml**
- Search for configrations for **ManagedIndexSchemaFactory** and remove them all.
- Add following configuration to the solrconfig.xml file
```xml
  <schemaFactory class="ClassicIndexSchemaFactory"/>
```
- Search for the following XML element and comment the whole element
```xml
  <processor class="solr.AddSchemaFieldsUpdateProcessorFactory">
```
- Rename the **[SOLR_CORE_PATH]/conf/managed-schema** to **schema.xml**
- Restart the solr instance and make sure the core is working without any issues.

Also You can have a look on the **solrconfig.xml** available in **solr_config** folder.

### Configuring the schema.xml

- Copy the schema/schema.xml to **TEMPLATE_FOLDER/search_configuration** folder as **solr.xml**.
The solr.xml is the renamed **managed-schema** file in the previous step, but it has additional changes to integrate
indexes made by haystack and your application.
- You can deploy the new schema by running the following command
```sh
   python manage.py build_solr_schema --filename=[SOLR base folder]/server/solr/[CORE_NAME]/conf/schema.xml && curl 'http://localhost:8983/solr/admin/cores?action=RELOAD&core=[CORE_NAME]&wt=json&indent=true'
```

Following are the changes
- The haystack related configurations , you can see this starting from the following comment
```xml
     <!--
    ######################## django-haystack specifics begin ########################
    -->
```
- The id field is removed from the original schema.xml , because it is already added by the django-haystack specfic configurations
```xml
<field name="id" type="string" indexed="true" stored="true" required="true" multiValued="false" />
```

### Issues and resolutions

##### Timestamp might fail when indexing

After changing to use SOLR6 if you encounter the following error for timestamp field
```error
Solr responded with an error (HTTP 400): [Reason: Invalid Date in Date Math String:
```
override the timestamp field in index to return the following format
```python
%Y-%m-%dT%H:%M:%SZ
```
Ex:
```python

import datetime
from haystack import indexes
from myapp.models import Note

class NoteIndex(indexes.SearchIndex, indexes.Indexable):

 postedDate = indexes.DateTimeField(model_attr='postedDate')

def prepare_postedDate(self,obj):
        return obj.postedDate.strftime('%Y-%m-%dT%H:%M:%SZ')
```

##### Failures in build_solr_schema or the curl command (Error 500 etc) or uses of solr itself

Check the solr log at ```[SOLR base folder]/server/logs/solr.log```
If you see errors related to **StopFilterFactory** with
**enablePositionIncrements** or if you see complaints about **SortableInfField**
then the problem is likely that you have a stale or incorrect config file where
 schema.xml should be.

**This can happen for two reasons:**
 * Mis-Named: haystack looks for **search_configuration/solr.xml NOT schema.xml**; ensure you
 have named it correctly
 * Unable-To-Find: Ensure that you have edited settings.py to include the
 appropriate TEMPLATES->DIRS entry so that haystack can find it.  

If it cannot find your template for any reason...it silently provides it's own solr5.x
 compatible one that is broken for solr6.x.
