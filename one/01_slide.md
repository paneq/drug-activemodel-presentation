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
