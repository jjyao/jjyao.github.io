---
layout: post
title: "WordPress XML-RPC API"
date: 2012-07-21 19:15
comments: true
categories: 
published: false
---

1. [User Authentication](#user_authentication)
1. [wp.getUsersBlogs](#wp.getUsersBlogs)
1. [wp.getAuthors](#wp.getAuthors)
1. [wp.getPost](#wp.getPost)
1. [wp.getPosts](#wp.getPosts)
1. [wp.newPost](#wp.newPost)
1. [wp.deletePost](#wp.deletePost)
1. [wp.getPostType](#wp.getPostType)
1. [wp.getPostTypes](#wp.getPostTypes)
1. [wp.getPostFormats](#wp.getPostFormats)
1. [wp.getPostStatusList](#wp.getPostStatusList)
1. [wp.getTaxonomy](#wp.getTaxonomy) 
1. [wp.getTaxonomies](#wp.getTaxonomies)
1. [wp.getTerm](#wp.getTerm)
1. [wp.getTerms](#wp.getTerms)
1. [wp.newTerm](#wp.newTerm)
1. [wp.editTerm](#wp.editTerm)
1. [wp.deleteTerm](#wp.deleteTerm)
1. [wp.getMediaItem](#wp.getMediaItem)
1. [wp.getMediaLibrary](#wp.getMediaLibrary)
1. [wp.uploadFile](#wp.uploadFile)
1. [wp.getCommentCount](#wp.getCommentCount)
1. [wp.getComment](#wp.getComment)
1. [wp.getComments](#wp.getComments)
1. [wp.newComment](#wp.newComment)
1. [wp.editComment](#wp.editComment)
1. [wp.deleteComment](#wp.deleteComment)
1. [wp.getCommentStatusList](#wp.getCommentStatusList)
1. [wp.getOptions](#wp.getOptions)
1. [wp.setOptions](#wp.setOptions)

<!-- more -->

## <a id="user_authentication"></a>User Authentication
WordPress自己提供的XML-RPC的API采用每次请求发送用户名和密码的方式来进行认证。默认由wp-xmlrpc-server的`login()`函数处理，这个方法又调用pluggable.php中的`wp_authenticate()`函数处理，而这个函数就会去执行'authenticate'对应的所有filter来进行用户认证。在这里WordPress自己有提供一个默认的filter是user.php中的`wp_authenticate_username_password()`函数，总的来说对于XML-RPC请求的用户认证主要就是由这个函数来完成的。

## <a id="wp.getUsersBlogs"></a>wp.getUsersBlogs
One WordPress can host multiple blogs. This request can retrieve list of blogs for this user.
#### Parameters
* **string username**
* **string password**

#### Return Values
* **array**
    * **struct**
        * **string blogid**
        * **string blogName**
        * **string url**
        * **string xmlrpc**: XML-RPC endpoint for the blog
        * **bool isAdmin**

``` php
$rpc_result = $wp_rpc->query('wp.getUsersBlogs', 'username', 'password');
/* Result */
Array ( [0] => Array ( [isAdmin] => 1 [url] => http://66.175.220.101/wordpress/ [blogid] => 1 [blogName] => 奇多科技有限公司 [xmlrpc] => http://66.175.220.101/wordpress/xmlrpc.php ) )
```

## <a id="wp.getAuthors"></a>wp.getAuthors
Retrieves a listing of all users assigned to a blog. ps: not just return Authors but all users, see [here](http://wordpress.org/support/topic/list-all-registered-users-authors-on-a-page-with-their-info)
#### Parameters
* **int blog_id**: If we only host one blog then the blog_id should be 1
* **string username**
* **string password**

#### Return Values
* **array**
    * **struct**
        * **string user_id**
        * **string user_login**
        * **string display_name**

#### Errors
* 401
    * If the user lacks the edit_posts cap(permission)

``` php
$rpc_result = $wp_rpc->query('wp.getAuthors', 1, 'jjyao', 'password');
/* Result */
Array ( [0] => Array ( [user_id] => 1 [user_login] => jjyao [display_name] => jjyao ) [1] => Array ( [user_id] => 2 [user_login] => kavin [display_name] => kavin ) [2] => Array ( [user_id] => 3 [user_login] => yyj [display_name] => yyj ) )
```

## <a id="wp.getPost"></a>wp.getPost
Retrieve a post of any registered post type
#### Parameters
* **int blog_id**: If we only host one blog then the blog_id should be 1
* **string username**
* **string password**
* **int post_id**
* **array fields**: Optional. List of field or meta-field names to include in response. If you omit this parameter, wordpress will include some default post fields for you. The default fields are 'post', 'terms', 'custom_fields'

#### Return Values
* **struct**: Note that the exact fields returned depends on the fields parameter
    * **string post_id**: post field
    * **string post_title**: post field
    * **datatime post_date**: post field
    * **datatime post_date_gmt**: post field
    * **datatime post_modified**: post field
    * **datatime post_modified_gmt**: post field
    * **string post_status**: post field
    * **string post_type**: post field
    * **string post_format**: post field
    * **string post_name**: post field
    * **int post_author**: post field
    * **string post_password**: post field
    * **string post_excerpt**: post field
    * **string post_content**: post field
    * **string link**: post field
    * **string comment_status**: post field
    * **string ping_status**: post field
    * **bool sticky**: post field
    * **struct post_thumbnail**: See [wp.getMediaItem](#wp.getMediaItem): post field
    * **array terms**: include post categories, tags, and other taxonomy: terms field
        * **struct**: See [wp.getTerm](#wp.getTerm)
    * **array custom_fields**: meta-data that you add to the post: custom_fields field
        * **struct**
            * **string id**
            * **string key**
            * **string value**
    * **struct enclosure**: They’re the bits of info in Atom and RSS feeds that make podcasting possible [reference](http://en.wikipedia.org/wiki/RSS_enclosure) : enclosure field
        * **string url**
        * **int length**
        * **string type**

#### Errors
* 401
    * If the user lacks the edit_posts cap(permission)
* 404
    * If no post with that post_id exists

#### Implementation
Firstly retrieve the given post data from the database(may use object cache), something like `SELECT * FROM wp_posts WHERE ID = post_id`. Then forward all retrieved data to `_prepare_post()`(class-wp-xmlrpc-server.php), return the result of `_prepare_post()`

``` php
$rpc_result = $wp_rpc->query('wp.getPost', 1, 'jjyao', 'password', 1, array('post', 'terms', 'custom_fields', 'enclosure'));
/* Result */
Array ( 
[post_id] => 1 
[post_title] => Hello world! 
[post_date] => IXR_Date Object ( [year] => 2012 [month] => 07 [day] => 18 [hour] => 06 [minute] => 15 [second] => 42 [timezone] => ) 
[post_date_gmt] => IXR_Date Object ( [year] => 2012 [month] => 07 [day] => 18 [hour] => 06 [minute] => 15 [second] => 42 [timezone] => ) 
[post_modified] => IXR_Date Object ( [year] => 2012 [month] => 07 [day] => 18 [hour] => 06 [minute] => 36 [second] => 56 [timezone] => ) 
[post_modified_gmt] => IXR_Date Object ( [year] => 2012 [month] => 07 [day] => 18 [hour] => 06 [minute] => 36 [second] => 56 [timezone] => ) 
[post_status] => publish 
[post_type] => post 
[post_name] => hello-world 
[post_author] => 1 
[post_password] => 
[post_excerpt] => 
[post_content] => Welcome to WordPress. This is your first post. Edit or delete it, then start blogging! 
[link] => http://66.175.220.101/wordpress/?p=1 
[comment_status] => open 
[ping_status] => open 
[sticky] => 
[post_thumbnail] => Array ( ) 
[post_format] => standard 
[terms] => Array ( [0] => Array ( [term_id] => 3 [name] => 奇多 [slug] => %e5%a5%87%e5%a4%9a [term_group] => 0 [term_taxonomy_id] => 3 [taxonomy] => category [description] => [parent] => 0 [count] => 1 ) ) 
[custom_fields] => Array ( ) 
[enclosure] => Array ( ) 
)
```

## <a id="wp.getPosts"></a>wp.getPosts
Retrieve list of posts of any registered post type and you have edit_post permission for each post. You can use some query conditions to filter some posts
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **struct filter**: Optional: an associative array
    * **string post_type**: If you don't specify the post_type, default is 'post', you can only specify one post type
    * **string post_status**: default is 'draft, publish, future, pending, private', you can specify several post status
    * **int number**: number of posts: default is 10
    * **int offset**: default is 0
    * **string orderby**: default is date(post_date column): allowed values include name(post_name column), author(post_author column), date(post_date column), title(post_title column), modified(post_modified column), menu_order(menu_order column), parent(post_parent column), ID(ID column), rand(RAND()), comment_count(comment_count column): **you can specify several values separated by space like 'date title', *but only one order can be specified*, so you can just get something like ORDER BY date, title ASC**
    * **string order**: default is DESC: you can only specify 'DESC' or 'ASC'
* **array fields**: Optional. See[wp.getPost](#wp.getPost)

#### Return Values
* **array**
    * **struct**: See[wp.getPost](#wp.getPost)

#### Notes
Response will only contain posts that the user has permission to edit. Therefore, there may be fewer than filter['number'] posts in the response.

#### Errors
* 401
    * If the user lacks the edit_posts cap(permission)

#### Implementation
Something like `SELECT * FROM wp_posts WHERE post_type = post_type AND (post_status = 'draft' OR post_status = 'publish' OR ...) LIMIT offset, number ORDER BY orderby order`

``` php
 $rpc_result = $wp_rpc->query('wp.getPosts', 1, 'jjyao', 'password', 
    array('post_type'=>'post', 'post_status'=>'publish, private', 'number'=>2, 'offset'=>0, 'orderby'=>'title date', 'order'=>'ASC') ,
    array('post', 'terms', 'custom_fields', 'enclosure'));
```

## <a id="wp.newPost"></a>wp.newPost
Create a new post of any registered post type.
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **struct content**
    * **string post_type**: default 'post'
    * **string post_status**: default 'draft'
    * **string post_title**: default empty string
    * **int post_author**: default current login user's ID
    * **string post_excerpt**: default empty string
    * **string post_content**: default empty string
    * **datetime post_date_gmt|post_date**: default is now
    * **string post_format**: default 'standard'
    * **string post_password**: default empty string
    * **string comment_status**: 'open' or 'closed': default value depends on option 'default_comment_status'
    * **string ping_status**: 'open' or 'closed': default value depends on option 'default_ping_status'
    * **bool sticky**: default false
    * **int post_thumbnail**: ID of a media item to use as the post thumbnail/featured image: empty value deletes, non-empty value adds/updates: A variable is considered empty if it does not exist or if its value equals FALSE
    * **int post_parent**: default 0
    * **array custom_fields**
        * **struct**
            * **string key**
            * **ststring value**
    * **struct terms**: Taxonomy names as keys, array of term IDs as values: terms specified by IDs must exist and belong to the given taxonomy
    * **struct terms_names**: Taxonomy names as keys, array of term names as values: the term may be ambiguous in a hierarchical taxonomy, in this case you should use term ID instead: by this parameter you can create new terms and assigned to a given taxonomy
    * **struct enclosure**
        * **string url**
        * **int length**
        * **string type**
    * **any other fields supported by wp_insert_post**

#### Return Values
* **string post_id**

#### Errors
* 401
    * If the user does not have the edit_posts cap for this post type
    * If user does not have permission to create post of the specified post_status
    * If post_author is different than the user's ID and the user does not have the edit_others_posts cap for this post type
    * If sticky=true and user does not have permission to make the post sticky
    * If a taxonomy in terms or terms_names is not supported by this post type
    * If terms or terms_names is set but user does not have assign_terms cap
    * If an ambiguous term name is used in terms_names
* 403
    * If invalid post_type is specified
    * If an invalid term ID is specified in terms
* 404
    * If no author with that post_author ID exists
    * If no attachment with that post_thumbnail ID exists

``` php
$rpc_result = $wp_rpc->query('wp.newPost', 1, 'jjyao', 'password', array(
'post_type'=>'post', 'post_status'=>'publish', 'post_title'=>'Test from mobile app',
'post_content'=>'Test from mobile app', 'comment_status'=>'closed',
'terms'=>array('category'=>array(1)), 'terms_names'=>array('post_tag'=>array('奇多')),
'post_date'=>new IXR_Date(time() - (7 * 24 * 3600))));
/* Result */
8
```

## <a id="wp.editPost"></a>wp.editPost
Edit an existing post of any registered post type
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **int post_id**
* **struct content**: See [wp.newPost](#wp.newPost) for valid set of fields. Only needs to contain fields that you wish to modify; all other fields will retain their current values.

#### Return Values
* **bool true**: If operation fails, won't return false, but Errors instead

#### Errors
* 404
    * If no post with that post_id exists

## <a id="wp.deletePost"></a>wp.deletePost
Delete an existing post of any registered post type.  
When the post and page is permanently deleted, everything that is tied to it is deleted also. This includes comments, post meta fields, and terms associated with the post.  
The post or page is moved to trash instead of permanently deleted unless trash is disabled, item is already in the trash. The attachment is moved to the trash instead of permanently deleted unless trash for media is disabled, item is already in the trash.   
If the post_type is hierarchical, point children of this post to its parent.   
The attachments associated with the post won't be deleted.
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **int post_id**

#### Return Values
* **bool true**

#### Errors
* 401
    * If the user does not have permission to delete the post
* 404
    * If no post with that post_id exists

``` php
$rpc_result = $wp_rpc->query('wp.deletePost', 1, 'jjyao', 'password', 10);
```

## <a id="wp.getPostType"></a>wp.getPostType
Retrieve a registered post type   
It is a stub

## <a id="wp.getPostTypes"></a>wp.getPostTypes
Retrieve list of registered post types    
It is a stub

## <a id="wp.getPostFormats"></a>wp.getPostFormats
Retrieve list of post formats   
It is a stub

## <a id="wp.getPostStatusList"></a>wp.getPostStatusList
Retrieve list of supported values for post_status field on posts   
It is a stub

## <a id="wp.getTaxonomy"></a>wp.getTaxonomy
Retrieve information about a taxonomy   
It is a stub

## <a id="wp.getTaxonomies"></a>wp.getTaxonomies
Retrieve a list of taxonomies   
It is a stub

## <a id="wp.getTerm"></a>wp.getTerm
Retrieve a taxonomy term
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **string taxonomy**
* **int term_id**

#### Return Values
* **struct**
    * **string term_id**
    * **string name**
    * **string slug**
    * **string term_group**
    * **string term_taxonomy_id**
    * **string taxonomy**
    * **string description**
    * **string parent**
    * **int count**
    
#### Errors
* 401
    * If the user does not have the assign_terms cap for this taxonomy
* 403
    * If invalid taxonomy name is specified
* 404
    * If no term with that term_id exists

``` php
$rpc_result = $wp_rpc->query('wp.getTerm', 1, 'jjyao', 'password', 'post_tag', 3);
/* Result */
Array ( [term_id] => 3 [name] => 奇多 [slug] => %e5%a5%87%e5%a4%9a [term_group] => 0 [term_taxonomy_id] => 3 [taxonomy] => post_tag [description] => [parent] => 0 [count] => 2 )
```

## <a id="wp.getTerms"></a>wp.getTerms
Retrieve list of terms in a taxonomy
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **string taxonomy**
* **struct filter**: Optional
    * **int number**
    * **int offset**
    * **string orderby**
    * **bool hide_empty**: Whether to return terms with count=0
    * **string search**: Restrict to terms with names that contain (case-insensitive) this value

#### Return Values
* **array**
    * **struct**: See [wp.getTerm](#wp.getTerm)

``` php
$rpc_result = $wp_rpc->query('wp.getTerms', 1, 'jjyao', 'password', 'category', array('search'=>'cate'));
/* Result */
Array ( [0] => Array ( [term_id] => 1 [name] => Uncategorized [slug] => uncategorized [term_group] => 0 [term_taxonomy_id] => 1 [taxonomy] => category [description] => [parent] => 0 [count] => 3 ) )
```

## <a id="wp.newTerm"></a>wp.newTerm
Create a new taxonomy term
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **struct content**
    * **string name**
    * **string taxonomy**
    * **string slug**: Optional
    * **string description**: Optional
    * **int parent**: Optional

#### Return Values
* **string term_id**

#### Errors
* 401
    * If the user does not have the manage_terms cap for this taxonomy
* 403
    * If invalid taxonomy name is specified
    * If the term name is empty
    * If parent is set but the taxonomy is not hierarchical
    * If no term with that parent ID exists

``` php
$rpc_result = $wp_rpc->query('wp.newTerm', 1, 'jjyao', 'password', array('name'=>'Dog', 'taxonomy'=>'post_tag', 'slug'=>'puppy'));
```

## <a id="wp.editTerm"></a>wp.editTerm
Edit an existing taxonomy term
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **struct content**
    * **string taxonomy**
    * **string name**: Optional
    * **string slug**: Optional
    * **string description**: Optional
    * **int parent**: Optional

#### Return Values
* **bool true**

#### Errors
* 401
    * If the user does not have the edit_terms cap for this taxonomy
* 403
    * If invalid taxonomy name is specified
    * If the term name is empty
    * If parent is set but the taxonomy is not hierarchical
    * If no term with that parent ID exists
* 404
    * If no term with that term_id ID exists

## <a id="wp.deleteTerm"></a>wp.deleteTerm
Delete an existing taxonomy term
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **string taxonomy**: You should provide the taxonomy because one term can belong to several taxonomies
* **int term_id**

#### Return Values
* **bool true**

#### Errors
* 403
    * If invalid taxonomy name is specified
    * If the user does not have the delete_terms cap for this taxonomy
* 404
    * If no term with that term_id ID exists 

## <a id="wp.getMediaItem"></a>wp.getMediaItem
Retrieve a media item (i.e, attachment)
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **int attachment_id**

#### Return Values
* **struct**
    * **datetime date_created_gmt**
    * **string parent**: ID of the parent post
    * **string link**
    * **string thumbnail**
    * **string title**
    * **string caption**
    * **string description**
    * **string metadata**
    * **string attachment_id**

#### Errors
* 403
    * If the user lacks the upload_files cap
* 404
    * If no attachment with that attachment_id exists

``` php
$rpc_result = $wp_rpc->query('wp.getMediaItem', 1, 'jjyao', 'password', 13); 
/* Result */
Array (
[attachment_id] => 13
[date_created_gmt] => IXR_Date Object ( [year] => 2012 [month] => 07 [day] => 27 [hour] => 03 [minute] => 44 [second] => 26 [timezone] => )
[parent] => 0
[link] => http://66.175.220.101/wordpress/wp-content/uploads/2012/07/favicon.png
[title] => favicon
[caption] => Octopress favicon
[description] => Octopress favicon
[metadata] => Array ( [width] => 16 [height] => 16 [hwstring_small] => height='16' width='16' [file] => 2012/07/favicon.png [image_meta] => Array ( [aperture] => 0 [credit] => [camera] => [caption] => [created_timestamp] => 0 [copyright] => [focal_length] => 0 [iso] => 0 [shutter_speed] => 0 [title] => ) )
[thumbnail] => http://66.175.220.101/wordpress/wp-content/uploads/2012/07/favicon.png )
```

## <a id="wp.getMediaLibrary"></a>wp.getMediaLibrary
Retrieve list of media items
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **struct filter**: Optional (and all members are optional)
    * **int number**: Total number of media items to retrieve: default is 5
    * **int offset**: default is 0
    * **int parent_id**: Limit to attachments on this post ID. 0 shows unattached media items. Unset shows all media items.
    * **string mime_type**: default is empty string: such as 'text/xml', 'text/plain', 'image/png', 'video/mp4'

#### Return Values
* **array**
    * **struct**: see [wp.getMediaItem](#wp.getMediaItem)

#### Errors
* 401
    * If the user lacks the upload_files cap

## <a id="wp.uploadFile"></a>wp.uploadFile
Upload a media file
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **struct data**
    * **string name**: Filename
    * **string type**: File MIME type
    * **string bits**: base64-encoded binary data
    * **bool overwrite**: Optional. Overwrite an existing attachment of the same name

#### Return Values
* **struct**
    * **string id**
    * **string file**: Filename
    * **string url**
    * **string type**

#### Errors
* 401
    * If the user does not have the upload_files cap
* 500
    * File upload failure

``` php
$rpc_result = $wp_rpc->query('wp.uploadFile', 1, 'jjyao', 'password', array('name'=>'nginx.pdf', 'type'=>'application/pdf', 'bits'=>new IXR_Base64(file_get_contents('/tmp/nginx.pdf')))); 
/* Result */
Array ( [id] => 22 [file] => nginx.pdf [url] => http://66.175.220.101/wordpress/wp-content/uploads/2012/07/nginx.pdf [type] => application/pdf )
```

## <a id="wp.getCommentCount"></a>wp.getCommentCount
Retrieve comment count for a specific post. post_trashed and trash comments won't be taken into consideration
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **string post_id**

#### Return Values
* **struct**
    * **int approved**
    * **int awaiting_moderation**
    * **int spam**
    * **int total_comments**

#### Errors
* 403
    * if the user does not have the edit_posts cap

``` php
$rpc_result = $wp_rpc->query('wp.getCommentCount', 1, 'jjyao', 'password', 1);
/* Result */
Array ( [approved] => 1 [awaiting_moderation] => 0 [spam] => 0 [total_comments] => 1 )
```

## <a id="wp.getComment"></a>wp.getComment
Retrieve a comment
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **int comment_id**

#### Return Values
* **struct**
    * **string comment_id**
    * **string parent**: When you reply a comment, your comment has a parent 
    * **string user_id**: 0 when the comment author isn't a registered user, otherwise the author should have a user_id in the system
    * **datetime dateCreated**
    * **string status**: possible value: 'hold'(awaiting_moderation), 'approve', 'spam', 'post_trashed', 'trash'
    * **string content**
    * **string link**
    * **string post_id**
    * **string post_title**
    * **string author**: author name
    * **string author_url**
    * **string author_email**
    * **string author_ip**
    * **string type**: comment has three types: regular comment, Trackback, Pingback [reference](http://codex.wordpress.org/Function_Reference/comment_type)

#### Errors
* 403
    * If the user does not have the moderate_comments cap
* 404
    * If no comment with that comment_id exists

``` php
$rpc_result = $wp_rpc->query('wp.getComment', 1, 'jjyao', 'password', 2);
/* Result */
Array(
[date_created_gmt] => IXR_Date Object ( [year] => 2012 [month] => 07 [day] => 30 [hour] => 01 [minute] => 45 [second] => 56 [timezone] => )
[user_id] => 0
[comment_id] => 2
[parent] => 0
[status] => approve
[content] => It is my comment
[link] => http://127.0.0.1/~jjyao/wordpress/?p=1#comment-2
[post_id] => 1
[post_title] => Hello world!
[author] => example
[author_url] => http://www.example.com
[author_email] => example@example.com
[author_ip] => 127.0.0.1
[type] => 
)
```

## <a id="wp.getComments"></a>wp.getComments
Retrieve list of comments
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **struct filter**
    * **int post_id**: default is empty: If empty, retrieve all comments
    * **string status**: default is 'approve or hold': possible value: 'hold'(awaiting_moderation), 'approve', 'spam', 'trash': there is no way to retrieve 'post-trashed' comments
    * **int number**: default is 10
    * **int offset**: default is 0

#### Return Values
* **array**
    * **struct**: see [wp.getComment](#wp.getComment) for fields

#### Errors
* 401
    * If the user does not have the moderate_comments cap

``` php
$rpc_result = $wp_rpc->query('wp.getComments', 1, 'jjyao', 'password', array('post_id'=>1, 'status'=>'approve', 'number'=>5, 'offset'=>0));
```

## <a id="wp.newComment"></a>wp.newComment
Create a new comment
#### Parameters
* **int blog_id**
* **string username**: if anonymous comment is allowed, the username and password parameters can be left as empty strings
* **string password**
* **int post_id**
* **struct comment**
    * **int comment_parent**: default is 0: specified when you reply a comment
    * **string content**
    * **string author**: you can omit the following three parameters if you provide the username and password
    * **string author_url**
    * **string author_email**

#### Return Values
* **int comment_id**

#### Errors
* 403
    * If anonymous comments are disallowed and invalid credentials are supplied
    * If comment does not follow required comment fields configuration
* 404
    * If no post with that post_id exists

``` php
$rpc_result = $wp_rpc->query('wp.newComment', 1, 'jjyao', 'password', 1, array('content'=>'comment from xmlrpc'));
```

## <a id="wp.editComment"></a>wp.editComment
Edit an existing comment
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **int comment_id**
* **struct comment**
    * **string status**: possible value: 'hold', 'approve', 'spam'
    * **datetime date_created_gmt**
    * **string content**
    * **string author**
    * **string author_url**
    * **string author_email**

#### Return Values
* **boolean true**

#### Errors
* 401
    * If status is not a valid comment status
* 403
    * If the user does not have the moderate_comments cap
    * If the user does not have permission to edit this comment
* 404
    * If no comment with that comment_id exists

``` php
$rpc_result = $wp_rpc->query('wp.editComment', 1, 'jjyao', 'password', 5, array('status'=>'spam', 'content'=>'modified comment from xmlrpc'));
```

## <a id="wp.deleteComment"></a>wp.deleteComment
Remove an existing comment
#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **int comment_id**

#### Return Values
* **boolean true**

#### Errors
* 403
    * If the user does not have the moderate_comments cap
    * If the user does not have permission to edit this comment
* 404
    * If no comment with that comment_id exists

## <a id="wp.getCommentStatusList"></a>wp.getCommentStatusList
Retrieve list of comment statuses  
It is a stub

## <a id="wp.getOptions"></a>wp.getOptions
Retrieve blog options

#### Parameters
* **int blog_id**
* **string username**
* **string password**
* **array options**: List of option names to retrieve. If omitted, all options will be retrieved

#### Return Values
* **array**
    * **struct**
        * **string desc**
        * **string value**
        * **bool readonly**

#### Errors
* This method will only return white-listed options. If a non-white-listed option is included in options, it will be omitted from the response

``` php
$rpc_result = $wp_rpc->query('wp.getOptions', 1, 'jjyao', 'password');
```

## <a id="wp.setOptions"></a>wp.setOptions
Edit blog options  
It is a stub
