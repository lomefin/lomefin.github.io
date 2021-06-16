---
layout: post
title: "Generate cookie file from Watir"
date: 2021-05-18 17:36:35 -0400
categories:
---

Disclaimer: this is the extended post from a [solution I gave on StackOverflow](https://stackoverflow.com/questions/67591606/how-can-i-generate-a-cookie-file-from-watir-cookies/67591607)

So I had to build a scraper which has to download a file.

It is pretty easy to download a file from a browser, set the download directory, navigate the page,  if the file is after a link, just click it.
But there is a catch, you can ask the browser to click the link and the download will follow, but how can you tell when its done, and if there are several files to download, which one is the latest?

Most of what I've seen is something like

    file_list = list_files_in_download_dir
    diff = []
    link.click
    10.times do
      file_list_now = list_files_in_download_dir
      diff = file_list_now - file_list
      break if diff.length.positive?
    end

    puts "The files are: #{diff}"

But having the file when it has been downloaded is top priority to me, maybe also to you.

So I guess another approach must be taken. I like to navigate using a browser, so maybe I could make the browser do must of the grunt work and when I'm ready I can download the file myself! But now I need to have the browser's cookie to identify myself as the proper user. So I had to take the cookies from Watir and pass them to [Typhoeus](https://github.com/typhoeus/typhoeus)

Since I use Typhoeus and Typhoeus wraps curl, from another article I learned that curl uses [Mozilla style cookie files](https://xiix.wordpress.com/2006/03/23/mozillafirefox-cookie-format/)

So, if your `Watir` instance is browser and the file to which you are going to write to is `file` you can do

    browser.cookies.to_a.each do |ch| 
      terms = []
      terms << ch[:domain]
      terms << ch[:same_site].nil? ? 'FALSE' : 'TRUE'
      terms << ch[:path]
      terms << ch[:secure] ? 'TRUE' : 'FALSE'
      terms << ch[:expires].to_i.to_s
      terms << ch[:name]
      terms << ch[:value]
      file.puts terms.join("\t")
    end
