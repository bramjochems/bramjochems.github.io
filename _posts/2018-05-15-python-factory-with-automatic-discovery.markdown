---
layout: post
title:  Factory with automatic discovery in python
date:   2018-05-15 21:05:56 +0200
categories: python
---

Recently, I needed to code (in python) a 'factory' class whilst not knowing when coding what all classes are that could potentially be instantiated. To make this a bit more concrete, I want to able to create a python package, that allows users to create probability distribution objects through a syntax such as:

```
create_distribution('lognormal')
```

The main thing I was looking to avoid here is that for every distribution created, users would have to revisit the factory function. The way I did this was by creating a abstract base class that distributions need to derive from and then have some code that recursively walks certain directories and checks for all classes that derive from this base class. The essence of the code (with things like checking for exceptions omitted), looked as follows.

The base class:

```
class Distribution(ABC):

	# Some details omitted that aren't really relevant
	
    @classmethod
    @abstractmethod
    def get_model_name(cls) -> str:
        pass

    @abstractmethod
    def pdf(self, value):
		pass
```

An example of a class deriving from this:

```
class LogNormal(Distribution):
	# Again, some details omitted
	
	@classmethod
    def get_model_name(cls):
        return 'lognormal'

	def pdf(self, value):
		return 0  # correctness of implementation isn't really relevant here
```

Now the factory itself is very easy, it just loads all classes that derive from Distribution into a dictionary and then looks up the dictionary:

```
def create_distribution(model_name, **params):
    global _DISTRIBUTIONS_LOADED
    global _DIST_DICT

    if not _DISTRIBUTIONS_LOADED:
        load_distributions()

    return _DIST_DICT[model_name](**params)  # error handling omitted
```

For efficiency reasons I chose to have the loading happen only a single time and keep track of whether things have been loaded already in a global variable. This is by no means strictly necessary, you could also write code where you call `load_distributions` on very call to `create_distribution` and then have the former return a dictionary.

The real work of course takes place in the `load_distributions` function:

```
def load_distributions():
    # again, simplified with no error handling etc
    global _DIST_DICT
    global _DISTRIBUTIONS_LOADED

	parent_name = '.'.join(__name__.split('.')[:-1]) or '__main__'
	parent_location = importlib.import_module(parent_name).__path__

	submodules_to_examine = import_submodules(parent_name, parent_location)
	for _, module in submodules_to_examine.items():
		for des, cls in inspect.getmembers(module, inspect.isclass):
			if issubclass(cls, Distribution) and not cls is Distribution:
				_DIST_DICT[cls.get_model_name()] = getattr(module, des)

	_DISTRIBUTIONS_LOADED = True
```

I'm sure that at least some of the code in `load_distributions` I actually found somewhere online (likely stackoverflow), but I don't recall where. Anyway, I was pleased with how this works and figured it might be useful enough to some, to share.

