require File.join(File.dirname(__FILE__), 'config', 'boot')

# -*- coding: iso-8859-1 -*-

require 'rubygems'
require 'rake'
require 'aws/s3'
require 'csv'


import File.join(SPREE_ROOT, 'Rakefile')

namespace :spree do
  desc "Export Products From Database to CSV File"
  task :export_products => :environment do
    require 'fastercsv'
    products = Product.find(:all)
    puts "Exporting to #{RAILS_ROOT}/tmp/products.csv" #some
    def make_really_pretty_taxon(leaf_taxon,pretty_taxon="")
      pretty_taxon = leaf_taxon.name + '/' + pretty_taxon if !leaf_taxon.nil?
      parent_taxon = Taxon.find(leaf_taxon.parent_id) if !leaf_taxon.nil? && !leaf_taxon.parent_id.nil?
      return pretty_taxon if parent_taxon.name == "Categories"  #This is the name of your root taxon. For this project it is "Shop By Categories"
      make_really_pretty_taxon(parent_taxon,pretty_taxon) if !parent_taxon.nil?
    end
    FasterCSV.open("#{RAILS_ROOT}/tmp/products.csv", "w") do |csv|

      csv << ["id", "sku", "name", "description","measure" ,"price" , "on_hand", "taxons", "has_variants", "no_of_variants", "deleted_at"] #mention all the columns which you need.

      products.each do |p|
        next if !p.deleted_at.nil?
        if !p.taxons[0].nil?
          really_pretty_taxon = make_really_pretty_taxon(p.taxons[0]).chomp!("/")    #Get the taxon associated with the product in a human readable format.
        end
        csv << [p.id,
                p.sku,
                p.name.titleize,
                p.description,
                nil,                       #this column is for option-types. Master variant does not have any.
                p.master_price.to_s,
                p.on_hand,
                really_pretty_taxon,
                p.has_variants?,
                p.variants.size,
                p.deleted_at
               ]

         if p.has_variants?
           puts p.name
           p.variants.each do |variant|
            puts  variant.sku
            if variant.option_values[0].nil?
              measure = nil
            else
              measure =  variant.option_values[0].presentation
            end
            csv << [variant.id,
                    variant.sku,
                    nil,                    #no name and description required for variants
                    nil,
                    measure,
                    variant.price.to_s,
                    variant.on_hand]
          end
        end
      end
    end
    puts "transferring to s3"  #If you are on heroku then its a good idea to transfer the generated file to amazon S3. Comment out otherwise.
    file = "#{RAILS_ROOT}/tmp/products.csv"
    AWS::S3::S3Object.store(file,open(file),'your-amazon-bucket')
    puts "Export Complete"
  end

  desc "Update / Import products from CSV File to Database, expects file=/path/to/import.csv"
  task :import_products => :environment do
    require 'fastercsv'

    n = 0
    u = 0

    product_with_variant = nil
    total_variants = nil
    FasterCSV.foreach(ENV['file']) do |row|
      id                = row[0]
      sku               = row[1].to_s.strip
      name              = row[2]
      description       = row[3].to_s.strip
      measure           = row[4].to_s.strip
      price             = row[5].to_s.strip
      on_hand           = row[6].to_s.strip
      taxon_string      = row[7].to_s.strip
      has_variants      = row[8].to_s.strip
      no_of_variants    = row[9].to_s.strip
      deleted_at        = row[10]

      if !total_variants.nil? && total_variants > 0 && !product_with_variant.nil?

        if id.nil?
          puts "found a new variant"
          puts sku
          puts price
          if sku.nil? || sku.eql?("")
            sku=product_with_variant.sku
          end
          variant = product_with_variant.variants.create(:sku => sku, :price => price.to_d)
          variant.option_values << OptionValue.find_by_presentation(measure) #here it assumes that the given option(in this case, weight) is already created in spree. You can change this as per your requirement
          variant.on_hand = on_hand.to_i
          variant.save!
        else
          puts "updating a variant"
          variant = product_with_variant.variants.find(id)
          variant.sku = sku
          variant.price = price.to_d
          puts "variant on hand #{on_hand.to_i}"
          variant.on_hand = on_hand.to_i
          variant.save!
        end
        total_variants -= 1
        if total_variants ==  0       #if this is the last variant
          product_with_variant.save!  #then save
          total_variants = nil
          product_with_variant = nil
          puts "saved with variants"
        end
        next
      end

      if id.nil? && !name.nil?   # a variant is not a new product. In the csv, the variant does not have the name specified.
        # Adding new product
        puts "Adding new product: #{sku}"
        product = Product.new()

        n += 1
      elsif !id.nil? && !name.nil?
        # Updating existing product

        next if id.downcase == "id"  #skip header row

        puts "Updating product: #{id}"
        product = Product.find(id)
        puts "Product found: #{product.name}"

        u += 1
      end

      next if name.nil?

      product.available_on =  Time.now
      taxonomy = Taxonomy.first          # default to main taxonomy for now
      if taxon_string.blank?
        puts "Warning: missing taxon info for #{name}"
      else
        taxon_to_store = taxonomy.root
        taxon_string.gsub('/',',').split(/\s*,\s*/).each do |singular_taxon|
          nxt = Taxon.find_or_create_by_name_and_parent_id_and_taxonomy_id(singular_taxon, taxon_to_store.id, taxonomy.id)
          taxon_to_store = nxt
        end
        product.taxons = [taxon_to_store]  #assumption: only one taxon for a product
      end

      if sku.nil? || sku.empty?
       # new_sku = rand(9999999) + 1000000;
         product.sku = "SOME" #You can write your own SKU generation code here.
      else
         product.sku = sku
      end

      product.name = name
      product.description = description

      product.master_price = price.to_d

      if has_variants.include? "true"
        puts "Yes we have some variants"
        total_variants = no_of_variants.to_i
        product_with_variant = product
        product.save!
      elsif has_variants.include? "false"
        product.save!
      end


    end

    puts ""
    puts "Import Completed - Added: #{n} | Updated #{u} Products"
   end
end
