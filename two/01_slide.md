!SLIDE code smaller
	@@@ ruby
	# smart_model.rb
	module SmartModel
		extend ::ActiveSupport::Concern

		included do
		  extend  ::ActiveModel::Naming
		  extend  ::ActiveModel::Translation
		  include ::ActiveModel::Conversion
		  include ::ActiveModel::Validations
		  include ::ActiveModel::MassAssignmentSecurity
		end

		def persisted?
		  false
		end

		def attributes=(values)
		  sanitize_for_mass_assignment(values).each do |k, v|
		    send("#{k}=", v)
		  end
		end

	end

!SLIDE code smaller
	@@@ ruby
	require 'active_record/connection_adapters/abstract/schema_definitions'
	module SearchByDates
	 extend ::ActiveSupport::Concern
	 DateConversion = ActiveRecord::ConnectionAdapters::Column.new(:anonymous, nil, :date)

	 def period=(v)
	  @period = v
	  return @period unless Date::RECOGNIZED_PERIODS.include?(@period)
	  self.valid_from = Date.calculate_start(v.to_sym)
	  self.valid_to = Date.calculate_end(v.to_sym)
	  return @period
	 end

	 def valid_from=(v)
	  @valid_from_before_type_cast = v
	  @valid_from = DateConversion.type_cast(v)
	 end

	 def valid_to=(v)
	  @valid_to_before_type_cast = v
	  @valid_to = DateConversion.type_cast(v)
	 end

	 included do
	  validates_inclusion_of :period, :in => Date::RECOGNIZED_PERIODS.map(&:to_s), 
	  :allow_blank => true, :allow_nil => true
	  if respond_to?(:attr_accessible)
	   attr_accessible :valid_from, :valid_to, :period
	  end
	 end

	end

!SLIDE code smaller
	@@@ ruby
	require 'smart_model'
	require 'search_by_dates'
	class Search

		include ::SmartModel
		include ::SearchByDates

		attr_accessor :producent, :brand, :color
		attr_accessible :producent, :brand, :color

		def initialize(attributes = {})
		  self.attributes = attributes
		end

		def results
		  scope = Car.scoped
		  if valid_from.present? && valid_to.present?
		   scope = scope.overlaps(valid_from, valid_to)
		  end
		  scope = scope.where(:producent => producent) unless producent
		  scope = scope.where(:brand => brand) unless brand.blank?
		  scope = scope.where(:color => color) unless color_blank?
		  return scope
		end

		def color_blank? # blank? or array of blanks
		  return color.to_a.reject{|p| p.blank? }.blank? if color.respond_to?(:to_a)
		  return color.blank?
		end
	end

!SLIDE code smaller
	@@@ ruby
	class CarsController < ApplicationController

		def search
		  @search = Search.new(params[:search])
		  if @search.valid?
		    @cars = @search.results.order("valid_from ASC")
		  else
		    @cars = Car.all # @cars = []
		  end
		  render :index
		end
	end

!SLIDE code smaller
	@@@
	<% content_for(:head) do %>
	 <%= render :partial => 'shared/js/calendar_selector' %>
	 <%= javascript_include_tag 'date_select_trio' %>
	<% end %>

	<%= semantic_form_for(@search, :url => search_cars_path) do |f| %>

	<%= f.inputs do %>
	 <%= f.input :color, :as => :check_boxes, :collection => Car.colors %>
	 <%= f.input :brand, :as => :select, :collection => Car.brands %>
	 <%= f.input :producent, :as => :select, :collection => Car.producents %>
	 <%= f.date_select_trio %>
	<% end %>

	<%= f.buttons %>
	<% end %>

!SLIDE code smaller
# Trzymanie w sesji (bez obiektów AR)
	@@@ ruby
	# http://www.ruby-forum.com/topic/173345
	# http://ruby-doc.org/core/classes/Marshal.html
	def marshal_dump
	 self.serializable_attributes.inject({}) do |hash, attr|
	  hash[attr] = send(attr)
	  hash
	 end
	end

	# http://www.ruby-forum.com/topic/173345
	# http://ruby-doc.org/core/classes/Marshal.html
	def marshal_load(hash)
	 hash.each do |ivar, value|
	  instance_variable_set(:"@#{ivar}", value)
	 end
	end
