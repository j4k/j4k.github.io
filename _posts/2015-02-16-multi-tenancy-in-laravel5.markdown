---
layout: post
title:  "Multi Tenancy in Laravel 5"
date:   2015-02-16 00:47:41
categories: application-architecture
---

Modern SaaS applications usually contain some form of multi tenancy in it's access control. A good example is [@slackhq][slack], which is business and it's staff centric.

[slack]: https://twitter.com/slackhq

### Types of multi tenancy

There are three main types of multi tenancy.

#### Separate database per tenant.

Probably the most secure, and the most expensive to maintain, each tenant get's their own copy of the database, has individual connection information, etc. This ensures that data from one client never gets mixed with another. On creation of a user account the database gets cloned and the connection information is stored against a master table containing all tenants. This is often viewed as the premium approach, as it is generally the most expensive in hardware and maintenance. Tenant customisation is easiest in this approach.

#### Same database, separate schemas

Some SQL implementations allow multi schemas in the same database (MySQL differs in this respect as `SCHEMA` and `DATABASE` are the same thing - this kind of set up in MySQL is achieved with [views][mysql-views] - but I'll save that for another post). This essentially lets you group up tables within the database that are tenant specific. Tenant customisation, to the data model like the separate database approach, is still fairly easy with this approach.

[mysql-views]: http://dev.mysql.com/doc/refman/5.6/en/create-view.html

A significant drawback to this approach is that tenant data is harder to restore in the event of a failure. If each tenant has it's own database, restoring a single tenant's data means simply restoring the database from the most recent backup. With this approach, restoring from that backup would mean overwriting all tenant data, even if some have experienced no data loss at all.

#### Same database, same schemas

The cheapest approach uses the same set of tables the host multiple tenants data, with a foreign key in every table that relates to a tenant id. This is cheapest to implement, and has the lowest hardware and backup costs, because it will allow you to serve the largest number of tenants per database server. However additional effort is needed in this approach to ensure data integrity between tenants.

-

### Tenant Considerations

The number, nature, and needs of the tenants in your application will affect the data architecture decision you make. If you are making something that contains very sensitive information, it would make sense to have some separation of database and data.



### How do shared databases multi tenant applications work?

Now that we've established the different set ups and paradigms, let's explore probably the easiest to set up within Laravel. The shared database, shared tables method.

In a typical SaaS set up, a client will authenticate and then be presented with their unique data, be it with an API, or a web page.

When the client authenticates, the application sets a global context that will scope all database queries to only return data for that current client. What this boils down to, is every query run with laravel's ORM will be scoped with an additional `WHERE` statement on models that need to be tenant specific. This approach is quite affective.

### Using the power of scopes

So Laravel 5 provides an interface that let's you modify the scope of an Eloquent Query Builder. The query builder is the underpinning of the ORM and Eloquent itself, turning Model::all's into SQL strings to be executed.

 This looks like this:

{% highlight php %}
<?php namespace Illuminate\Database\Eloquent;

interface ScopeInterface {

	/**
	 * Apply the scope to a given Eloquent query builder.
	 *
	 * @param  \Illuminate\Database\Eloquent\Builder  $builder
	 * @param  \Illuminate\Database\Eloquent\Model  $model
	 * @return void
	 */
	public function apply(Builder $builder, Model $model);

	/**
	 * Remove the scope from the given Eloquent query builder.
	 *
	 * @param  \Illuminate\Database\Eloquent\Builder  $builder
	 * @param  \Illuminate\Database\Eloquent\Model  $model
	 *
	 * @return void
	 */
	public function remove(Builder $builder, Model $model);

}
{% endhighlight %}

Implementing this on a custom scoping class, it is quite easy to create a custom scoping that executes on every request for an eloquent model, combined with a trait for models that are required to be scoped at every step. Finally, let's stick an override in that let's us query without the scope being enabled.

Let's start with the Scoping Class, which we will call `TenantScope`. To keep this as succinct as possible, we will assume one tenantID is required, but this is trivial to implement for multiple tenants per query.

{% highlight php %}
<?php namespace Acme/Scoping;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\ScopeInterface;

class TenantScope implements ScopeInterface {

	private $model;

	protected $tenant_col = 'tenant_id';

	public function apply(Builder $builder, Model $model)
	{
		$builder->whereRaw($model->getTenantWhereClause($this->tenant_col, 1));
	}

	public function remove(Builder $builder, Model $model)
	{
		$query = $builder->getQuery();
		foreach( (array) $query->wheres as $key => $where) {
			if($this->isTenantConstraint($model, $where, $this->tenant_col, 1)) {
				unset($query->wheres[$key]);

				$query->wheres = array_values($query->wheres);
				break;
			}
		}
	}

	public function isTenantConstraint($model, array $where, $tenantColumn, $tenantId)
	{
		return $where['type'] == 'raw' && $where['sql'] == $model->getTenantWhereClause($tenantColumn, $tenantId);
	}

}
{% endhighlight %}

This class is fairly simple, and all it does is apply an additional where based on our tenant_col class property to each query, and remove it when it needs to be removed. The apply method here gets called in `Eloquent\Model` in the `applyGlobalScopes` method, which gets executed after every query in the `newQuery` method.

The remove method is in place for a query to be run without a global scope, which is useful when you want to run a query one time without a tenant scope.

### A trait!

{% highlight php %}
<?php namespace Acme/Scoping/Traits;

use App;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Database\Eloquent\Model;
use Acme\Scoping\TenantScope;

trait TenantScopedModelTrait
{
	 public static function bootTenantScopedModelTrait()
     {
        $tenantScope = App::make('Acme\Scoping\TenantScope');

        // Add Global scope that will handle all operations except create()
        static::addGlobalScope($tenantScope);
    }

    public static function allTenants()
    {
        return with(new static())->newQueryWithoutScope(new TenantScope());
    }

    public function getTenantWhereClause($tenantColumn, $tenantId)
    {
        return "{$this->getTable()}.{$tenantColumn} = '{$tenantId}''";
    }

}

{% endhighlight %}

This is the basic Trait that is ready to go onto Eloquent models. The boot method applies our scope to the model that has this trait applied. The `allTenants` method simply returns a new `$builder` object without a global scope added to it, to query all tenants information.

Now all that needs to done is to apply this trait to an eloquent model that needs to be scoped by tenant.

{% highlight php %}
<?php namespace App;

use Illuminate\Database\Eloquent\Model;

class TenantData extends Model {

	use TenantScopedModelTrait;

	protected $table = 'tenant_data';

}
{% endhighlight %}

### Wrapping it up

This is a fairly basic implementation. Moving forward it might be smart to cater for more than one tenant column identifier, and add some exceptions in to make it clear when a model is missing, whether it's due to our global scoping, or just because the model doesn't exist. This could probably be done on the trait.

Next steps would be to set up something like nginx wildcard for subdomains of your application, and use the subdomain as a query scope.







