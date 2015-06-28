---
layout: page
status: publish
published: true
title: 'Feeling Fuzzy: Name Matching at the MPC'
author:
  display_name: Fran Fabrizio
  login: fran
  email: fran@umn.edu
  url: ''
author_login: fran
author_email: fran@umn.edu
excerpt: "The MPC's data has been cited thousands&nbsp;of times by publications as
  varied as the academic journal Demography to the New York Times. &nbsp;In this article,
  we explore how we make connections between those citations and the user accounts
  in our data dissemination products to help connect users to their work and to others
  doing similar work.\r\n\r\n"
wordpress_id: 171
wordpress_url: http://tech.popdata.org/?p=171
date: '2014-10-10 18:04:38 -0500'
date_gmt: '2014-10-10 18:04:38 -0500'
categories:
- Data Processing
- Featured
tags:
- Blarticle
comments: []
---
<p>The MPC's data has been cited thousands&nbsp;of times by publications as varied as the academic journal Demography to the New York Times. &nbsp;In this article, we explore how we make connections between those citations and the user accounts in our data dissemination products to help connect users to their work and to others doing similar work.</p>
<p><a id="more"></a><a id="more-171"></a>Here at the MPC, one of the ways that we measure the impact of our work is by tracking publications which cite our data. &nbsp;This helps us understand how our work is impacting research areas such as global health, the welfare of women and children, immigration patterns and socioeconomic changes. &nbsp;Our data show up in a lot of different types of publications, including newspapers, online media, PhD dissertations, governmental policy documents and non-profit charity reports, but the majority of citations are in the form of academic journal articles. &nbsp;We track these in our internal bibliography database. &nbsp;Our website users are obligated to cite us when they user our data in a publication, but they don&rsquo;t need to directly report their publications to us, so we rarely learn about citations from the authors. &nbsp;Rather, our user support staff proactively searches for citations to our data on the web and within the common journal article repositories and conference proceedings. &nbsp;We then enter them directly into our database. &nbsp;We&rsquo;ve done this for many years now and so far we&rsquo;ve found more than 11,000 citations to our work.</p>
<p>We&rsquo;ve always wanted to make links between the users using our data dissemination websites and the authors of the citations in our database. &nbsp;Logic says that authors who are citing our data are very likely to also have a user account with us which they used to retrieve the data in the first place. &nbsp;If we could make those links, we&rsquo;d be able to display a user&rsquo;s citations as part of their profile page in the webapp. &nbsp;This might encourage our users to let us know about additional publications of theirs that have used our data. &nbsp;We&rsquo;d be able to use these links to help users find other users who are doing similar work and publishing on similar topics.</p>
<p>To make these links, we&rsquo;d have to find a way to match records between the two databases. &nbsp;We have the user&rsquo;s first and last name from their account registration, so we can attempt to match that to the authors on each citation. &nbsp;I recently tackled this task, and thought it was a good peek into the sorts of data munging we end up doing around here.</p>
<p>Name matching is a problem with lots of wrinkles. &nbsp;People make typos when entering their account registration info or submitting citations to our database. &nbsp;People get married and change their name. &nbsp;Accented characters get modified or stripped by some citation systems. &nbsp;Middle names or initials are included or excluded. &nbsp;And so on. &nbsp;Any matching strategy would have to be more complex than an exact string match on the names.</p>
<p>Luckily, we&rsquo;re no strangers to fuzzy matching around here. &nbsp;When you deal with 200-year-old census data that was originally gathered by hand and digitized decades or centuries later, you come to expect dirty data. &nbsp;We&rsquo;ve written some pretty sophisticated code to try to cluster census strings like birthplace and occupation so that common typos, omissions, and transpositions don&rsquo;t result in a whole new occupation or place of birth. &nbsp;For instance, &ldquo;laborer&rdquo; and &ldquo;laborrer&rdquo;, or &ldquo;Philadelphia&rdquo; and &ldquo;Philadelpha&rdquo;. &nbsp;We&rsquo;ve even tackled harder cases such as &ldquo;farmer&rdquo; and &ldquo;framer&rdquo;. &nbsp;Since both could be valid, we can look at the surrounding context (Do they reside on a farm or in a city? How is the rest of their family employed?) to determine the most likely truth.</p>
<p>But this name matching was just a side project, and I was looking for a quick-and-dirty approach. &nbsp;I had one day to play with this problem, and I wanted to get this longstanding todo item off of our plate. &nbsp;Even if I only uncovered some of the actual linkages, that represents potentially thousands of new links which didn&rsquo;t exist before. &nbsp;That would kickstart our plans to enrich our user profiles and help our users network with each other. &nbsp;In this case, great certainly seemed like the enemy of good.</p>
<p>When quick-and-dirty is the order of the day, I typically turn to Perl. &nbsp;And I knew that Perl had a wealth of string comparison modules already written, a fact I hoped would save me precious time. The first class of modules I turned to were those which implemented phonetic string comparison algorithms.</p>
<h3>Phonetic String Comparison Algorithms</h3><br />
The idea with phonetic algorithms is to produce a code from the input string that encodes similar-sounding input strings into the same code. &nbsp;For instance, the Soundex algorithm will encode both &ldquo;Jonathan" and &ldquo;Johnathan&rdquo; as &ldquo;J535". &nbsp;This is accomplished via a set of rules that get applied to each character of the input string to distill it down to its phonetic code.</p>
<p>Soundex was the first popular phonetic algorithm, dating all the way back to 1918, and is still very popular today. &nbsp;Most major programming languages and databases have a Soundex implementation. &nbsp;I&rsquo;ve used Soundex for similar problems at prior jobs, but it wasn&rsquo;t until I was researching this article that I realized that it was our very own industry that led to Soundex&rsquo;s popularity! &nbsp;It was used in the 1930s for a retrospective analysis of the 1890-1920 US Censuses.</p>
<p>Soundex is good enough for many tasks, but because Soundex carries the first letter of the input string to the code, it misses some obvious cases. &nbsp;For instance, &ldquo;Catherine" encodes to C365 while &ldquo;Katherine" encodes to K365. &nbsp;Also, Soundex is designed for English names and does not handle surnames from other languages very well.</p>
<p>There have been a number of attempts to improve upon these deficiencies over the years. &nbsp;The NYSIIS algorithm was designed to perform better with hispanic surnames, while Daitch-Mokotoff was an attempt to improve performance for Slavic and Yiddish surnames, and K&ouml;lner attempted the same for words of Germanic origin. &nbsp;The Metaphone algorithm came along in 1990 as an improvement on the original Soundex algorithm, but was still focused on English words.</p>
<p>Since our users come from all over the world, none of these specialized algorithms were a perfect fit. &nbsp;I was looking for one which was optimized to handle lots of language origins. &nbsp;Luckily, in 2000 the author of Metaphone, Lawrence Philips, published Double Metaphone, which features a more complicated encoding scheme. &nbsp;The &ldquo;Double&rdquo; refers to the fact that the algorithm returns not one but two codes, a primary and a secondary, in order to account for ambiguities and surnames which share a common ancestor. &nbsp;Double Metaphone was also designed to account for the quirks and nuances of strings of many different origins. &nbsp;It seems we had a winner. &nbsp;Perl has the Text::DoubleMetaphone implementation, and this is what I chose to use for this project.</p>
<p>This algorithm proved to be great for catching many common cases of fuzzy name matches. &nbsp;The table below has some examples of name pairs which produce a phonetic match using the Double Metaphone approach.</p>
<p>[table width ="30%" style ="" responsive ="false"]<br />
[table_head]<br />
[th_column]String 1[/th_column]<br />
[th_column]String 2[/th_column]<br />
[/table_head]<br />
[table_body]<br />
[table_row]<br />
[row_column]Gabriel Aquino[/row_column]<br />
[row_column]Gabrial Aquino[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Albert Esteve[/row_column]<br />
[row_column]Alberto Esteve[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Leticia Marteleto[/row_column]<br />
[row_column]Let&iacute;cia Marteleto[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Rodolfo Gutierrez[/row_column]<br />
[row_column]Rodolfo Guti&eacute;rrez[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Hoyt Bleakley[/row_column]<br />
[row_column]Hoty Bleakley[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Alex Levkov[/row_column]<br />
[row_column]Alexey Levkov[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Jeffrey Lin[/row_column]<br />
[row_column]Jeffery Lin[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]FlorenceMae Waldron[/row_column]<br />
[row_column]Florence Mae Waldron[/row_column]<br />
[/table_row]<br />
[/table_body]<br />
[/table]</p>
<p>However, phonetic matching isn&rsquo;t perfect. &nbsp;It can miss some cases, such as these real-world examples from our system which do not produce a phonetic match but are, in fact, referring to the same person:</p>
<p>[table width ="30%" style ="" responsive ="false"]<br />
[table_head]<br />
[th_column]String 1[/th_column]<br />
[th_column]String 2[/th_column]<br />
[/table_head]<br />
[table_body]<br />
[table_row]<br />
[row_column]Olivier Deschenes[/row_column]<br />
[row_column]Oliver Deschenes[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Martin Fieder[/row_column]<br />
[row_column]Martin Fielder[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Edward Levine[/row_column]<br />
[row_column]Edward Lewine[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Keisuke Okada[/row_column]<br />
[row_column]Eisuke Okada[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Melissa Moreno[/row_column]<br />
[row_column]Melissa Gearhart Moreno[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Jeronimo Muniz[/row_column]<br />
[row_column]Jeronimo Oliveira Muniz[/row_column]<br />
[/table_row]<br />
[/table_body]<br />
[/table]</p>
<p>Phonetic approaches can also generate some head-scratching false positives, such as the ones in the table below.</p>
<p>[table width ="100%" style ="" responsive ="false"]<br />
[table_head]<br />
[th_column]String&nbsp;1[/th_column]<br />
[th_column]String&nbsp;2[/th_column]<br />
[/table_head]<br />
[table_body]<br />
[table_row]<br />
[row_column]Christine O&rsquo;Leary [/row_column]<br />
[row_column]Christopher H. Wheeler[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Jie Wu[/row_column]<br />
[row_column]Zhou Yu[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Donna Leicach[/row_column]<br />
[row_column]Daniel Koch[/row_column]<br />
[/table_row]<br />
[/table_body]<br />
[/table]</p>
<p>Observing this, I then wanted to apply a second algorithmic approach to act as a check-and-balance to the phonetic algorithm.</p>
<h3>String Distance Algorithms</h3><br />
The second approach I wanted to explore was a string distance algorithm. &nbsp;A string distance is simply a representation of how many edits (typically insertions, deletions and substitutions) would be required to transform one string into another - known as the &ldquo;edit distance". &nbsp;I had been familiar with this concept from prior work in bioinformatics, where it is common to categorize how alike two sequences of DNA are based on their edit distance from one another (pairwise alignment). &nbsp;I wanted to include a distance algorithm to catch some of the cases where a phonetic algorithm would miss the match.</p>
<p>Perhaps the earliest popular algorithm for string distance is the Hamming distance algorithm, developed in the 1950s and used in telecommunications to indicate the number of flipped bits in two binary sequences, as a measure of signal error. &nbsp;The Hamming distance algorithm only considers substitutions, so it is limited to words of equal length. &nbsp;We&rsquo;d need something more flexible than that. &nbsp;The Levenshtein algorithm, one of the most ubiquitous distance algorithms, allows insertions and deletions along with substitutions, and therefore is a promising alternative. &nbsp;Damerau-Levenshtein is a refinement of Levenshtein which also allows transpositions to count as a single edit instead of as a delete-and-insert combo in the original Levenshtein. &nbsp;Since transposition is a common typo when entering text by hand, Damerau-Levenshtein was an attractive choice. &nbsp;Perl provides implementations for all of these, including Text::Levenshtein::XS and Text::Levenshtein::Damerau::XS. &nbsp;I chose to proceed with the latter (for now - read on).</p>
<p>There&rsquo;s a lot more nuance with interpreting edit distance results vs. phonetic algorithm results. &nbsp;The phonetic approach is pretty cut and dry; either two strings have the same phonetic code or they don&rsquo;t. &nbsp;With edit distances, it&rsquo;s up to the user to determine how a distance should be interpreted. &nbsp;The only generality that one can hope to conclude is that two strings with a smaller distance are more likely to be a match (although even here there are edge cases which violate this principle). &nbsp;But where do we draw the quality line? &nbsp;At some edit distance N, you&rsquo;re going to get into a gray area where you have enough confidence&nbsp;that something is a match to consider it, but not enough confidence that it is a match to automatically accept it.</p>
<p>So a critical question to ask is &ldquo;What are the range of distances within which my confidence of a match is such that it&rsquo;s worth further review?&rdquo; &nbsp;This is largely a trial-and-error phase of the problem - you run some trial data and see where the rate of actual matches falls off relative to the distance calculation. &nbsp;To encapsulate this idea for the user support personnel who were ultimately going to review the list of potential matches I produced, I wanted to produce a confidence score for each match that factored in the results from both the phonetic algorithm and the distance algorithm.</p>
<h3>Representing Match Confidence</h3><br />
I reasoned about my match&nbsp;confidence algorithm by considering all the possibilities:</p>
<ol>
<li>If two strings have a phonetic match and a distance of zero, they are a match with 100% confidence. &nbsp;(Identical strings)</li>
<li>If two strings have a phonetic match and a non-zero distance, my confidence should decrease as distance increases. &nbsp;(Similar strings that sound the same)</li>
<li>If two strings do not have a phonetic match but have a non-zero distance, my confidence should decrease more rapidly as distance increases. &nbsp;(Similar strings that don&rsquo;t sound the same)</li>
<li>If two strings do not have a phonetic match and have a zero distance, something has gone horribly wrong. &nbsp;(Two identical strings that don&rsquo;t sound the same?!)</li><br />
</ol><br />
Case #1 is clear - if the two strings are exactly the same, you&rsquo;ve got a match by definition. &nbsp;And&nbsp;case #4 should never happen. &nbsp;Luckily, my code has yet to produce an instance of case #4, so I&rsquo;ve passed at least one sanity test!</p>
<p>Cases #2 and #3 are where all the fun and danger lurks.</p>
<p>At this point, an important observation should be considered: not all distances are created equal. &nbsp;Compare&nbsp;these two distance calculations:</p>
<pre><strong>&ldquo;bear&rdquo; and &ldquo;beet&rdquo; => 2</strong></pre></p>
<pre><strong>&ldquo;cheeseburger&rdquo; and &ldquo;cheesebruger&rdquo; => 2</strong></pre><br />
Despite that fact that one pair of words is a lot more related than the other, the distance calculations are identical. &nbsp;Intuitively, a distance of 2 between two words of length 12 is a lot more impressive than a distance of 2 between two words of length 4. &nbsp;I wanted to represent this in my confidence somehow, so I created a simple ratio calculation:</p>
<pre><strong>ratio = distance / length of the longer string</strong></pre><br />
I use this ratio in two ways. &nbsp;First, if the ratio is too high&nbsp;(meaning the number of differing characters is approaching the total length of the strings), I toss out the comparison as a non-match immediately by comparing the ratio to&nbsp;a "distance match threshold". &nbsp;For my input data, 0.1 turns out to be an effective threshold - above that and almost all of the extra potential matches turned out to be false positives.</p>
<p>The second way I use the ratio is as the modifier in case #2&rsquo;s confidence calculation. &nbsp;So now for case #2 I chose a simple match&nbsp;confidence formula:</p>
<pre><strong>confidence = 1 - ratio</strong></pre><br />
As the distance increases, my ratio will be a bigger number and my confidence will decrease. &nbsp;For smaller words, it will take fewer edits to make the confidence really plummet than it does for longer words. &nbsp;To say this is a poor excuse for an elegant algorithm is an understatement&hellip;but remember, quick and dirty. &nbsp;This captures the intended effect, if in a bit of a &ldquo;blunt-force instrument&rdquo; kind of way.</p>
<p>Case #2 also exposes some interesting false positives, such as the ones I discussed in the phonetic algorithm section above. &nbsp;Those string comparisons all produce phonetic matches, yet have distance scores so poor their ratios are above the threshold to even be considered a potential distance match. &nbsp;So here the distance approach is acting as an effective check-and-balance to the phonetic approach, as desired.</p>
<p>Case #3 is the most interesting - strings that don&rsquo;t sound the same but are close in distance. &nbsp;To wit:</p>
<p>[table width ="30%" style ="" responsive ="false"]<br />
[table_head]<br />
[th_column]String 1[/th_column]<br />
[th_column]String 2[/th_column]<br />
[/table_head]<br />
[table_body]<br />
[table_row]<br />
[row_column]Olivier Deschenes[/row_column]<br />
[row_column]Oliver Deschenes[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Roberto Hernandez[/row_column]<br />
[row_column]Roberto Fernandez[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Martin Fieder[/row_column]<br />
[row_column]Martin Fielder[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Edward Levine[/row_column]<br />
[row_column]Edward Lewine[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Keisuke Okada[/row_column]<br />
[row_column]Eisuke Okada[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Ying Fang[/row_column]<br />
[row_column]Ying Yang[/row_column]<br />
[/table_row]<br />
[/table_body]<br />
[/table]</p>
<p>These name pairs all fail to match phonetically but have a distance score of 1. &nbsp;This is the most difficult bunch to sort out automatically - even after human inspection there are several cases that probably need further investigation. &nbsp;As distance scores increase, the likelihood of a true match falls off precipitously. &nbsp;In fact, in my testing I determined it was only worth looking at name pairs in this category with a distance of 1 or 2. &nbsp;So, to reflect this in my confidence score for case #3:</p>
<p><strong>confidence = 1 - ratio - (0.1 * distance^2)</strong></p>
<p>I still consider the ratio because it&rsquo;s a lot easier for two short, completely unrelated names to have a small distance than it is for two longer names. As for the distance penalty, trial and error determined that 0.1 * distance^2 was an appropriate penalty. &nbsp;There was more intuition than science involved here, and it could certainly be tweaked more, but it achieved the desired effect of stratifying the possible matches well away from the ones almost certain to be non-matches. &nbsp;And yes, this can result in a negative confidence score in some limited cases. &nbsp;Remember, quick and dirty.</p>
<p>My results were getting better all the time, but there was one last group of names that was still bugging me. &nbsp;It seemed to me that not all edits are created equal. &nbsp;For example, these two name pairs both have an edit distance of 9:</p>
<p>[table width ="30%" style ="" responsive ="false"]<br />
[table_head]<br />
[th_column]String 1[/th_column]<br />
[th_column]String 2[/th_column]<br />
[th_column]Edit Distance[/th_column]<br />
[/table_head]<br />
[table_body]<br />
[table_row]<br />
[row_column]Guillaume Boucher[/row_column]<br />
[row_column]Guillaume Vandenbroucke[/row_column]<br />
[row_column]9[/row_column]<br />
[/table_row]<br />
[table_row]<br />
[row_column]Melissa Moreno[/row_column]<br />
[row_column]Melissa Gearhart Moreno[/row_column]<br />
[row_column]9[/row_column]<br />
[/table_row]<br />
[/table_body]<br />
[/table]</p>
<p>I felt that there ought to be some way that I could penalize the 9 edits it takes to go from Boucher -> Vandenbroucke more severely than the 9 edits it takes to insert Melissa&rsquo;s maiden (or middle) name Gearhart. &nbsp;The presence/absence of a middle initial or name is a common case for us, but because these name pairs rarely phonetically match, and the distance calculations tended to be high, we were missing these types of matches.</p>
<p>Distance algorithms generally assign each type of edit the same cost. &nbsp;But what if I could explicitly state that I wanted insertions and deletions to be less penalized than substitutions? &nbsp;I went searching for an algorithm that allowed me to do this and came across Text::Levenshtein::Flexible. &nbsp;This module by Matthias Bethke is an extension of the Levenshtein algorithm which allows the user to assign costs for insertion, deletion and substitution. &nbsp;Perfect!</p>
<p>After some trial and error, I settled upon keeping insertions and deletions at 1 while penalizing substitutions with a distance score of 4 ([1,1,4] is my edit penalty array). &nbsp;I had to remember to update my ratio calculation as well. &nbsp;I now use:</p>
<pre><strong>ratio = distance / 4 * length of longer string &nbsp;</strong></pre><br />
I have to do this because now my average distance score is going to be higher, and I want to make sure that my ratio doesn&rsquo;t cause lots of good matches to cross over the threshold for being discarded as a potential distance match. &nbsp;(If the Text::Levenshtein::Flexible module allowed for non-integer edit costs, I could have specified [0.25, 0.25, 1] instead and then left the ratio calculation unaltered.)</p>
<p>This change had the desired effect. &nbsp;Comparisons like Melissa Moreno <=> Melissa Gearhart Moreno which strictly involved insertions were no longer being penalized as strongly as Guillaume Boucher <=> Guillaume Vandenbroucke.</p>
<h3>Wrapping Up<a href="http://tech.popdata.org/wp-content/uploads/2014/09/haster.jpg"><img class="size-medium wp-image-227 alignright" src="http://tech.popdata.org/wp-content/uploads/2014/09/haster-300x217.jpg" alt="haster" width="300" height="217" /></a></h3><br />
At this point, I was happy with my algorithm soup. &nbsp;I ran the process over the entire set of over 100k users and 10k citations and by the end of the day I had forwarded a list of potential matches and their confidence scores to our user support team for verification. &nbsp;Preliminary indications are that we should be able to make links for at least 50% of our citations. &nbsp;Not bad for a day&rsquo;s work!</p>
<p>Another job crossed off the list. &nbsp;It wasn&rsquo;t always pretty, and there&rsquo;s certainly room for improvement, but this was a job that called for the 80/20 rule and I&rsquo;m pretty pleased with how it turned out. &nbsp;I hope others find this article useful for your own fuzzy name matching challenges, and I also hope that it gives you a small window into the sorts of tasks we tackle day in and day out here at MPC IT!</p>
