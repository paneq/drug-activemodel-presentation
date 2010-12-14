!SLIDE title-slide
# Active Model #

!SLIDE bullets incremental
# Plan #

* Przegląd wybranych funkcjonalności ActiveModel
* Pokaz aplikacji

!SLIDE code small

	@@@ ruby
	# activemodel/lib/active_model/validator.rb
	class Person
	 include ActiveModel::Validations
	 validates_with MyValidator
	end

	class MyValidator < ActiveModel::Validator
	 def validate(record)
	  if some_complex_logic
	   record.errors[:base] = "This record is invalid"
	  end
	 end

	 private
	  def some_complex_logic
	  end
	end

!SLIDE 
Thread-safe ?

!SLIDE code smaller
	@@@ ruby
	# The easiest way to add custom validators 
	# for validating individual attributes
	# is with the convenient
	# ActiveModel::EachValidator for example:

	class TitleValidator < ActiveModel::EachValidator
	
	 def validate_each(record, attribute, value)
	  unless ['Mr.', 'Mrs.', 'Dr.'].include?(value)
		 record.errors[attribute] << 'must be Mr. Mrs. or Dr.' 
	  end
	 end
	
	end

!SLIDE code small
	@@@ ruby
	# This can now be used in combination
	# with the +validates+ method
	class Person
	 include ActiveModel::Validations
	 attr_accessor :title

	 validates :title, :presence => true, :title => true
	end

!SLIDE code smaller
	@@@ ruby
	# Validator may also define a +setup+ 
	# instance method which will get called
	# with the class that using that validator
	# as it's argument. This can be
	# useful when there are prerequisites 
	# such as an attr_accessor being present
	# for example:
	#
	class MyValidator < ActiveModel::Validator
	 def setup(klass)
	  klass.send :attr_accessor, :custom_attribute
	 end
	end

!SLIDE code smaller
	@@@ ruby
	class EachValidator < Validator
	 attr_reader :attributes

	 # Returns a new validator instance. All options will be available via the
	 # +options+ reader, however the <tt>:attributes</tt> option will be removed
	 # and instead be made available through the +attributes+ reader.
	 def initialize(options)
	  @attributes = Array.wrap(options.delete(:attributes))
	  raise ":attributes cannot be blank" if @attributes.empty?
	  super
	  check_validity!
	 end

	 # Override this method in subclasses with the validation logic, adding
	 # errors to the records +errors+ array where necessary.
	 def validate_each(record, attribute, value)
	  raise NotImplementedError
	 end

	 # Hook method that gets called by the initializer allowing verification
	 # that the arguments supplied are valid. You could for example raise an
	 # ArgumentError when invalid options are supplied.
	 def check_validity!
	 end
	end

!SLIDE code smaller
	@@@ ruby
	# BlockValidator is a special EachValidator which receives a block on initialization
	# and call this block for each attribute being validated. +validates_each+ uses this
	# Validator.
	class BlockValidator < EachValidator
		def initialize(options, &block)
		  @block = block
		  super
		end

		private

		def validate_each(record, attribute, value)
		  @block.call(record, attribute, value)
		end
	end

!SLIDE
Problem z BlockValidator gdy chcemy się dostać do opcji


!SLIDE
validations.rb

!SLIDE code smaller
	@@@ ruby
	# == Active Model Validations
	#
	# Provides a full validation framework to your objects.
	#
	# A minimal implementation could be:
	#
	#   class Person
	#     include ActiveModel::Validations
	#
	#     attr_accessor :first_name, :last_name
	#
	#     validates_each :first_name, :last_name do |record, attr, value|
	#       record.errors.add attr, 'starts with z.' if value.to_s[0] == ?z
	#     end
	#   end
	#
	# Which provides you with the full standard validation
	# stack that you
	# know from Active Record:
	#
	#   person = Person.new
	#   person.valid?                   # => true
	#   person.invalid?                 # => false
	#
	#   person.first_name = 'zoolander'
	#   person.valid?                   # => false
	#   person.invalid?                 # => true
	#   person.errors                   # => #<OrderedHash>


!SLIDE code small
	@@@ruby
	module Validations
	 extend ActiveSupport::Concern
	 include ActiveSupport::Callbacks

	 included do
	  extend ActiveModel::Translation

	  attr_accessor :validation_context
	  define_callbacks :validate, :scope => :name

	  class_attribute :_validators
	  self._validators = Hash.new { |h,k| h[k] = [] }
	end

!SLIDE code smaller
	@@@ ruby
	#   class Person
	#     include ActiveModel::Validations
	#
	#     attr_accessor :first_name, :last_name
	#
	#     validates_each :first_name, :last_name do |record, attr, value|
	#       record.errors.add attr, 'starts with z.' if value.to_s[0] == ?z
	#     end
	#   end
	#
	# Options:
	# * <tt>:on</tt> - Specifies when this validation is active (default is
	#   <tt>:save</tt>, other options <tt>:create</tt>, <tt>:update</tt>).
	#
	# * <tt>:allow_nil</tt> - Skip validation if attribute is +nil+.
	#
	# * <tt>:allow_blank</tt> - Skip validation if attribute is blank.
	#
	# * <tt>:if</tt> - Specifies a method, proc or string to call to determine
	#   if the validation should occur (e.g. <tt>:if => :allow_validation</tt>,
	#   or <tt>:if => Proc.new { |user| user.signup_step > 2 }</tt>). The method,
	#   proc or string should return or evaluate to a true or false value.
	#
	# * <tt>:unless</tt>
	def validates_each(*attr_names, &block)
	 options = attr_names.extract_options!.symbolize_keys
	 validates_with BlockValidator, options.merge(:attributes => attr_names.flatten), &block
	end

!SLIDE code small
	@@@ ruby 
	class Comment
	 include ActiveModel::Validations

	 validate :must_be_friends
	
	 validate do |comment|
	  comment.must_be_friends
	 end

	 def must_be_friends
	  unless commenter.friend_of?(commentee)
	   errors.add(:base, "Must be friends to leave a comment")
	  end
	 end
	

!SLIDE code smaller
	@@@ ruby
	# Returns the Errors object that holds all information about attribute error messages.
	def errors
	 @errors ||= Errors.new(self)
	end

	# Runs all the specified validations and returns true if no errors were added
	# otherwise false. Context can optionally be supplied to define which callbacks
	# to test against (the context is defined on the validations using :on).
	def valid?(context = nil)
	end

	# Performs the opposite of <tt>valid?</tt>. Returns true if errors were added,
	# false otherwise.
	def invalid?(context = nil)
	 !valid?(context)
	end

!SLIDE code smaller
	@@@ ruby
	# Hook method defining how an attribute value should be retrieved.
	# By default this is assumed to be an instance named
	# after the attribute. Override this method in subclasses should
	# you need to retrieve the value for a given attribute differently: 
	alias :read_attribute_for_validation :send
	
	   class MyClass
	     include ActiveModel::Validations
	
	     def initialize(data = {})
	       @data = data
	     end
	
	     def read_attribute_for_validation(key)
	       @data[key]
	     end
	   end

!SLIDE
 errors.rb

!SLIDE code smaller
	# When passed a symbol or a name of a method, returns an array of errors
	# for the method.
	#
	#   p.errors[:name]   # => ["can not be nil"]
	#   p.errors['name']  # => ["can not be nil"]
	def [](attribute)
	end

	# Adds to the supplied attribute the supplied error message.
	#
	#   p.errors[:name] = "must be set"
	#   p.errors[:name] # => ['must be set']
	def []=(attribute, error)
	end

	# Iterates through each error key, value pair in the error messages hash.
	# Yields the attribute and the error for that attribute.  If the attribute
	# has more than one error message, yields once for each error message.
	#
	#   p.errors.add(:name, "can't be blank")
	#   p.errors.add(:name, "must be specified")
	#   p.errors.each do |attribute, errors_array|
	#     # Will yield :name and "can't be blank"
	#     # then yield :name and "must be specified"
	#   end
	def each
	end

!SLIDE code smaller
	def add_on_empty(attributes, options = {})
	end

	def add_on_blank(attributes, options = {})
	end

!SLIDE code smaller
	# Translates an error message in its default scope
	# (<tt>activemodel.errors.messages</tt>).
	#
	# Error messages are first looked up in <tt>models.MODEL.attributes.ATTRIBUTE.MESSAGE</tt>,
	# if it's not there, it's looked up in <tt>models.MODEL.MESSAGE</tt> and if that is not
	# there also, it returns the translation of the default message
	# (e.g. <tt>activemodel.errors.messages.MESSAGE</tt>). The translated model name,
	# translated attribute name and the value are available for interpolation.
	#
	# When using inheritance in your models, it will check all the inherited
	# models too, but only if the model itself hasn't been found. Say you have
	# <tt>class Admin < User; end</tt> and you wanted the translation for
	# the <tt>:blank</tt> error +message+ for the <tt>title</tt> +attribute+,
	# it looks for these translations:
	#
	# <ol>
	# <li><tt>activemodel.errors.models.admin.attributes.title.blank</tt></li>
	# <li><tt>activemodel.errors.models.admin.blank</tt></li>
	# <li><tt>activemodel.errors.models.user.attributes.title.blank</tt></li>
	# <li><tt>activemodel.errors.models.user.blank</tt></li>
	# <li>any default you provided through the +options+ hash (in the activemodel.errors scope)</li>
	# <li><tt>activemodel.errors.messages.blank</tt></li>
	# <li><tt>errors.attributes.title.blank</tt></li>
	# <li><tt>errors.messages.blank</tt></li>
	# </ol>
	def generate_message(attribute, type = :invalid, options = {})
	end


!SLIDE
 naming.rb

!SLIDE code smaller
	@@@ ruby
	class Name < String
		attr_reader :singular, :plural, :element, :collection, :partial_path, :i18n_key
		alias_method :cache_key, :collection

		def initialize(klass)
		  super(klass.name)
		  @klass = klass
		  @singular = ActiveSupport::Inflector.underscore(self).tr('/', '_').freeze
		  @plural = ActiveSupport::Inflector.pluralize(@singular).freeze
		  @element = ActiveSupport::Inflector.underscore(ActiveSupport::Inflector.demodulize(self)).freeze
		  @human = ActiveSupport::Inflector.humanize(@element).freeze
		  @collection = ActiveSupport::Inflector.tableize(self).freeze
		  @partial_path = "#{@collection}/#{@element}".freeze
		  @i18n_key = ActiveSupport::Inflector.underscore(self).tr('/', '.').to_sym
		end

		# Transform the model name into a more humane format, using I18n. By default,
		# it will underscore then humanize the class name
		#
		#   BlogPost.model_name.human # => "Blog post"
		#
		# Specify +options+ with additional translating options.
	    def human(options={})
	     return @human unless @klass.respond_to?(:lookup_ancestors) &&
		                       @klass.respond_to?(:i18n_scope)

	     I18n.(...)
	    end
	end

!SLIDE code smaller
	@@@ ruby
	# == Active Model Naming
	#
	# Creates a +model_name+ method on your object.
	#
	# To implement, just extend ActiveModel::Naming in your object:
	#
	#   class BookCover
	#     extend ActiveModel::Naming
	#   end
	#
	#   BookCover.model_name        # => "BookCover"
	#   BookCover.model_name.human  # => "Book cover"
	#
	#   BookCover.model_name.i18n_key              # => "book_cover"
	#   BookModule::BookCover.model_name.i18n_key  # => "book_module.book_cover"
	#
	# Providing the functionality that ActiveModel::Naming provides in your object
	# is required to pass the Active Model Lint test.  So either extending the provided
	# method below, or rolling your own is required..
	module Naming
		# Returns an ActiveModel::Name object for module. It can be
		# used to retrieve all kinds of naming-related information.
		def model_name
		  @_model_name ||= ActiveModel::Name.new(self)
		end
	end

!SLIDE code smaller
	@@@ ruby
	# conversion.rb
	# Handles default conversions: to_model, to_key and to_param.
	#
	#   class ContactMessage
	#     include ActiveModel::Conversion
	#
	#     # ContactMessage are never persisted in the DB
	#     def persisted?
	#       false
	#     end
	#   end
	module Conversion
		def to_model
		  self
		end

		def to_key
		  persisted? ? [id] : nil
		end

		def to_param
		  persisted? ? to_key.join('-') : nil
		end
	end


!SLIDE code smaller
	@@@ ruby
	# translation.rb
	#
	#   class TranslatedPerson
	#     extend ActiveModel::Translation
	#   end
	#
	#   TranslatedPerson.human_attribute_name('my_attribute')
	#   # => "My attribute"
	module Translation

		# Returns the i18n_scope for the class. Overwrite if you want custom lookup.
		def i18n_scope
		  :activemodel
		end

		# When localizing a string, it goes through the lookup returned by this
		# method, which is used in ActiveModel::Name#human,
		# ActiveModel::Errors#full_messages and
		# ActiveModel::Translation#human_attribute_name.
		def lookup_ancestors
		  self.ancestors.select { |x| x.respond_to?(:model_name) }
		end

		# Transforms attribute names into a more human format
		#   Person.human_attribute_name("first_name") # => "First name"
		def human_attribute_name(attribute, options = {})
		end
	end


