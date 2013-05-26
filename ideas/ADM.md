Automatic 
Dynamic
Pragmatic

Event catcher
Action taker

Document creation
Document modification
Document deletion
Document organizer

Secretary

SecretaryBot

DocumentBot

DocBot
NodeBot

Document manager

Declarative Automatic Document Manager (DADM)
Configurable Document Organizer (CDO)

Configuration Language

Auto Filer

Automatic Document Manager
Automatic Document Handler




Use cases
---------

Simply automatically file document of specified class in specified "directory"

````
$x->catch('create-or-update')
    ->ofClass('/Cms/Foobar/Document/Foo')
    ->fileUnder('/cms/foobar');
````

Create site:

````
$x->catch('create')
    ->at($x->path()->childOf('/sites'))
    ->ofClass('/Cms/Foobar/Document/Site')
    ->will(array(
        $x->do('touch-document')
            ->from('new')
            ->ofClass('Cms/Document/Foobar')
            ->named('foobar')
            ->at('./')
            ->mapProperties(array(
                'fooProp' => 'barfoo',
                'barProp' => function (Context $c) {
                    return $c->getDocument()->getFoobar().' bar';
                },
            ))
        ,
        $x->do('touch-document')
            ->from('template')
            ->withName('foobar_template')
            ->withParameters(array(
                'foobar' => function (Context $c) {
                    return $c->getDocument()->getSomeParameter();
                },
                'barfoo' => 'something',
            ))
            ->named(function(Context $c) { 
                return $c->getDocument()->getTitle().' foo';
            })
            ->at('./foobar')
        ,
    ))
;

$x->catch('delete')
    ->at($this->path()->descendant('/sites'))
    ->ofClass('/Cms/Foobar/Document/Site')
    ->will(array(
        $x->do('delete-document')
            ->at($this->path(function (Context $c) {
                return $this->getDocument()->getUser()->getId();
            }))
    ))
;
````

Auto Routing

````

// automatically create a route for a blog post
$x->catch('create')
    ->at($this->path()->any())
    ->ofClass('/Cms/Blog/Document/Post')
    ->will(array(
        $x->do('touch-document')
            ->updateReferrers()
            ->fromTemplate('cmf.routing.route')
            ->named(function (Context $c) {
                $slugifier = $c->getContainer()->get('slugifier');
                return $slugifier->slugify($c->getDocument->getTitle());
            }),
            ->at($this->path(function (Context $c) {
                $slugifier = $c->getContainer()->get('slugifier');
                return sprintf(
                    '/cms/routes/blog/%s/posts',
                    $slugifier->slugify($this->getDocument()->getBlog()->getTitle())
                );
            })),
    ));

// leave a redirect route when a route changes
$x->catch('update')
    ->when('path-changed') // [path-changed|name-changed|directory-changed]
    ->at($this->path()->any())
    ->ofClass('/Cms/Routing/Document/Route')
    ->will(array(
        $x->do('touch-document')
            ->fromTemplate('cmf.routing.redirect')
            ->withParameters(array(
                'redirect_route' => function (Context $c) {
                    return $c->getDocument()->getRoute();
                },
            ))
            ->at(function (Context $c) {
                return $c->getOriginalDocument()->getId();
            })
    ));
````

Multisite signup:

````

// catch site orders
$x->catch('create')
    ->ofClass('/Cms/Document/SiteOrder')
    ->putAt('/cms/site-orders') // automatically set the path
    ->requires(array(
        'list', 'of', 'required', 'fields'
    ))
    ->will(array(

        // create site object
        $x->do('create-document')
            ->setStackAlias('site')
            ->fromTemplate(function (Context $c) { 
                return $c->getDocument()->getSiteTemplate();
            })
            ->withParameters(array(
                'site_title' => function ($c) { 
                    return $c->getDocument()->getSiteTitle();
                },
                'host' => function ($c) {
                    return $c->getDocument()->getSiteHost();
                },
            ))
            ->named(function ($c) {
                return $c->getDocument()->getSiteHost();
            })
            ->at('/cms/sites'),

        // create user
        $x->do('create-document')
            ->setStackAlias('user')
            ->fromTemplate('my_cms.user')
            ->withParameters(array(
                'username' => function ($c) {
                    return $c->getDocument()->getUsername();
                },
                'password' => function ($c) {
                    $enc = $c->getContainer()->get('security.encoder');
                    return $enc->encode($c->getDocument()->getPlainPassword());
                },
                'sites' => function ($c) {
                    return array(
                        $c->getStack()->findByAlias('site')
                    );
                },
            )),

        // mirror skeleton site
        $x->do('mirror')
            ->from(function ($c) {
                return '/cms/platforms/'.$c->getDocument()->getSiteTemplate();
            })
            ->to(function ($c) {
                return $c->getStack()->findByAlias('site')->getId();
            }),
    ));
````

And in code land:


````
// explicitly enable the service - we don't want this beast
// running all the time!
$xManager->enable('signup');

$order = new SiteOrder;
$order->setSiteTitle('Dans Site');
$order->setSiteHost('www.example.com');
$order->setUsername('foobar');
$order->setPlainPassword('this-wont-be-stored');

$dm->persist($order);
$dm->flush();
`````

Note that we do not have to set the prefix, it is done automatically by `->putAt()'
