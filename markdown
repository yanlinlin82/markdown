#!/usr/bin/perl -w
use strict;
use warnings;
use File::Basename;
use Digest::MD5 qw(md5_hex);

my $version = "0.1.0";
$version .= " ($1)" if '$Id$' =~ m/Id: (.{1,7}).* /;

my $in_dir = ".";
my $disable_section_numbers = 0;
my $disable_toc = 0;
my $disable_embedding = 0;
my $verbose = 0;

my $article_title = "";
my @section_titles = ();
my %urls = ();

my %escape_table = ();
foreach my $char (split //, '\\`*_{}[]()>#+-.!') {
	$escape_table{$char} = md5_hex($char);
}

sub TrimLines
{
	my $text = shift;
	$text =~ s/\A\n+//; # remove leading empty lines
	$text =~ s/\s+\z//; # remove tailing spaces
	return $text;
}

sub Tables {
	my $text = shift;

	$text =~ s{
		^
		[ \t]{0,3}
		\|([ \t]*(.+)[ \t]*\|)+
		[ \t]*
		\n            # table header

		[ \t]{0,3}
		\|(((\:?)\-+(\:?)\|)+)
		[ \t]*
		\n            # table header line

		(
			(
				[ \t]{0,3}
				\|((.*?)\|)+
				[ \t]*
				\n
			)+
		)             # table content
	}{
		my $header = $1;
		my $line = $3;
		my $content = $7;
		my $th = TableHeader($header, $line);
		my $tr = TableContent($content, $line);
		"<table>\n" . $th . $tr . "</table>";
	}egmx;

	return $text;
}

sub TableHeader
{
	my $text = shift;
	my $line = shift;

	my $result = '';
	while ($text =~ /[ \t]*(.+?)[ \t]*\|/g) {
		my $header = CodeSpans($1);
		$line =~ /(\:?)\-+(\:?)\|/g;
		if ($1 and $2) {
			$result .= '<th align="center">';
		} elsif ($1) {
			$result .= '<th align="left">';
		} elsif ($2) {
			$result .= '<th align="right">';
		} else {
			$result .= '<th>';
		}
		$result .= $header . '</th>';
	}

	return "<tr>$result</tr>\n";
}

sub TableContent
{
	my $text = shift;
	my $line = shift;

	my @align = ();
	while ($line =~ /(\:?)\-+(\:?)\|/g) {
		if ($1 and $2) {
			push @align, ' align="center"';
		} elsif ($1) {
			push @align, ' align="left"';
		} elsif ($2) {
			push @align, ' align="right"';
		} else {
			push @align, '';
		}
	}

	my $result = '';
	while ($text =~ /[ \t]*\|(((.*?)\|)+)[ \t]*\n/mg) {
		my $line = $1;
		$result .= '<tr>';
		my $index = 1;
		while ($line =~ /[ \t]*(.*?)[ \t]*\|/g) {
			my $text = CodeSpans($1);
			if (exists $align[$index]) {
				$result .= "<td" . $align[$index] . ">$text</td>";
			} else {
				$result .= "<td>$text</td>";
			}
			$index += 1;
		}
		$result .= "</tr>\n";
	}

	return $result;
}

sub CodeSpans
{
	my $text = shift;

	$text =~ s@
			(`+)		# $1 = Opening run of `
			(.+?)		# $2 = The code block
			(?<!`)
			\1			# Matching closer
			(?!`)
		@
			my $c = "$2";
			$c =~ s/^[ \t]*//g; # leading whitespace
			$c =~ s/[ \t]*$//g; # trailing whitespace
			$c = EncodeCode($c);
			"<code>$c</code>";
		@egsx;

	return $text;
}

sub Outdent
{
	my $text = shift;
	$text =~ s/^[ ]{1,4}//mg;
	return $text;
}

sub EncodeCode {
	my $text = shift;
	$text =~ s{&}{&amp;}g;
	$text =~ s{<}{&lt;}gx;
	$text =~ s{>}{&gt;}gx;

	$text =~ s{\*}{$escape_table{'*'}}g;
	$text =~ s{_}{$escape_table{'_'}}g;
	return $text;
}

my @levels = (0, 0, 0, 0, 0, 0, 0);

sub Header
{
	my ($title, $level) = @_;
	$title = CodeSpans($title) if $title;
	if ($level < 1 or $level > 6) {
		return $title;
	}

	$levels[$level]++;
	@levels[($level + 1)..6] = 0 if $level < 6;

	my $id = "";
	if ($level == 1) {
		$article_title = $title unless $article_title;
	} else {
		$id = join(".", @levels[2..$level]);
		if (not $disable_section_numbers) {
			$title = "$id $title";
		}
	}

	if (not $disable_toc) {
		if ($level >= 2) {
			push @section_titles, "<li class=\"level_" . ($level - 1)
					. "\"><a href=\"#$id\">$title</a></li>";
		}
	}
	return "<h$level><a name=\"$id\">$title</a></h$level>";
}

sub Sections
{
	my $text = shift;
	my $header_pattern = qr{
		(
			\n(\#{1,6})[ ]+(.+)
		|
			\n[ ]{0,3}(.+)[ ]*\n[ ]{0,3}(=+|\++)
		)
		\s*
		(?=\n)
	}mx;

	my @parts = split(/$header_pattern/, "\n$text\n");
	my @sections;
	push @sections, Paragraphs(shift @parts);
	while (@parts) {
		shift @parts; # skip first match
		my $leading = shift @parts;
		my $title1 = shift @parts;
		my $title2 = shift @parts;
		my $type = shift @parts;
		my $content = shift @parts;
		my $level = $leading ? length($leading) : ($type =~ /=/ ? 1 : 2);
		my $title = $leading ? $title1 : $title2;
		push @sections, Header($title, $level);
		push @sections, Paragraphs($content);
	}
	$text = TrimLines(join("\n", @sections));
    $text = "<section>\n$text\n</section>" if $text;
	return $text;
}

sub Paragraphs
{
	my $text = shift;

	my $pattern_code_or_list = qr{
		^(
			[ ]{4}              # 4 spaces
		|
			[ ]{0,3}[*+-][ ]    # unordered list
		|
			[ ]{0,3}[0-9]+\.[ ] # ordered list
		)
		.*\n
	}mx;

	# fill spaces in empty lines of code block and/or list
	$text =~ s{
		($pattern_code_or_list)
		(\n+)
		(?=$pattern_code_or_list)
	}{
		$1 . (("    \n") x length($3));
	}mgex;

	my @parts = split(/\n\n+/, "$text\n\n");
	return TrimLines(join("\n", map{Paragraph($_)}@parts));
}

my $list_item = qr{
    ^[ ]{0,3}      # leading (0-3) spaces
    (
        ([*+-])    # leading character: '#', '+' or '-'
    |
        ([0-9]+\.) # leading number (with a dot)
    )
    \s+            # followed by at least one space
    (.*)           # list item text
    \s*\n          # tailing spaces and EOL
}mx;

sub Lists
{
	my $text = shift;
    my @parts = split(/$list_item/, "$text\n");
    shift @parts; # skip first empty match
    my $tag = "";
    $text = "";
    while (@parts) {
        my $leading = shift @parts;
        my $ul = shift @parts;
        my $ol = shift @parts;
        my $item = shift @parts;
        my $content = shift @parts;
        if (not $tag) {
            $tag = $ul ? "ul" : "ol";
        }
		$item .= "\n\n" . Outdent($content) if $content;
        $text .= "<li>" . Paragraphs($item) . "</li>\n";
    }
    $text = "<$tag>$text</$tag>" if $tag;
	return $text;
}

my @url_names = ();

sub Links
{
	my $text = shift;
	my @parts = split(/\n[ ]{0,3}\[(.*)\]:\s*(.*)\s*(?=\n)/, "\n$text\n");
	$text = shift @parts;
	while (@parts) {
		my $name = shift @parts;
		my $link = shift @parts;
		$urls{$name} = $link;
		$text .= shift @parts;
	}
	$text =~ s{
		(\!?)
		\[([^]]*)\]
		(
			\(([^)]*)\)
		|
			\[([^]]*)\]
		)
	}{
		my $is_img = $1;
		my $alt = $2;
		my $src = $4;
		if (not $src) {
			my $name = $5;
			if ($name) {
				push @url_names, $name;
				$src = "url://" . (scalar(@url_names) - 1);
			}
		}
		my $res = "";
		if ($is_img) {
			if (not $src and not $alt) {
				$res = "<img alt=\"invalid image\">";
			} else {
				$res = "<img src=\"$src\"";
				$res .= " alt=\"$alt\"" if $alt;
				$res .= ">";
			}
		} else {
			if (not $src and not $alt) {
				$res = "<mark>invalid link</mark>";
			} else {
				$res = "<a href=\"" . ($src ? $src : $alt) . "\">";
				$res .= ($alt ? $alt : $src);
				$res .= "</a>";
			}
		}
	}mgex;
	return $text;
}

sub Block
{
	my $text = shift;

	# Horizontal lines
	$text =~ s{^[ ]{0,2}([ ]?\*[ ]?){3,}[ \t]*$}{\n<hr>\n}gmx;
	$text =~ s{^[ ]{0,2}([ ]? -[ ]?){3,}[ \t]*$}{\n<hr>\n}gmx;
	$text =~ s{^[ ]{0,2}([ ]? _[ ]?){3,}[ \t]*$}{\n<hr>\n}gmx;

	$text = Tables($text);
	$text = Links($text);
	$text = CodeSpans($text);

	# Emphasis
	$text =~ s{(\*\*|__)(.+?[*_]*)\1}{<strong>$2</strong>}mg;
	$text =~ s{(\*|_)(.+?[*_]*)\1}{<em>$2</em>}mg;

	return TrimLines($text);
}

sub CodeBlock
{
	my $text = shift;
	return "<pre><code>" . TrimLines(EncodeCode(Outdent($text))) . "</code></pre>";
}

sub DumpParts
{
    my $prefix = shift @_;
    my $text = shift @_;
    print "===== $prefix =====\n";
    print "whole = '$text'\n";
    print "----- " . scalar(@_) . " parts -----\n";
    my $i = 0;
    for my $ss (@_) {
        print "part[$i] = " . (defined $ss ? "'$ss'" : "undef") . "\n";
        ++$i;
    }
}

sub ParagraphSepByList
{
    my $text = shift;
    if ($text) {
        my @parts = split(/\n\n(([ ]{4,}.*\n)+)(?=\n)/, "\n\n$text\n\n");
        my $s = shift @parts;
		$text = "";
		$text = Block($s) if $s;
		$text = "<p>$text</p>";
        while (@parts) {
            $text .= CodeBlock(shift @parts);
            shift @parts;
            $s = shift @parts;
			if ($s) {
				$text .= "<p>" . Block($s) . "</p>";
			}
        }
    }
	return $text;
}

sub Paragraph
{
	my $text = shift;

    my $list_pattern = qr{
        (
            (
                $list_item
                (              # following text lines
                    ^[ ]{4,}   # leading at least 4 spaces
                    .*\n
                )*
            )+
            (?=\n)
        )
    }mx;

    my @parts = split(/$list_pattern/, "\n$text\n\n");
    $text = ParagraphSepByList(shift @parts);
    while (@parts) {
        $text .= Lists(shift @parts);
        shift @parts; # skip four matches
        shift @parts;
        shift @parts;
        shift @parts;
        shift @parts;
        shift @parts;
        $text .= ParagraphSepByList(shift @parts);
    }
    return $text;
}

sub Base64EncodeFile {
	my $code = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/';
	my $filename = shift;
	open FILE, $filename or return 0;
	binmode(FILE);
	my ($buf, $data, $n);
	while (($n = read FILE, $data, 3) != 0) {
		if ($n == 3) {
			my $a = ord(substr($data, 0, 1));
			my $b = ord(substr($data, 1, 1));
			my $c = ord(substr($data, 2, 1));
			$buf .= substr($code, $a / 4, 1);
			$buf .= substr($code, ($a % 4) * 16 + ($b / 16), 1);
			$buf .= substr($code, ($b % 16) * 4 + ($c / 64), 1);
			$buf .= substr($code, $c % 64, 1);
		} elsif ($n == 2) {
			my $a = ord(substr($data, 0, 1));
			my $b = ord(substr($data, 1, 1));
			$buf .= substr($code, $a / 4, 1);
			$buf .= substr($code, ($a % 4) * 16 + ($b / 16), 1);
			$buf .= substr($code, ($b % 16) * 4, 1);
			$buf .= '=';
		} else {
			my $a = ord(substr($data, 0, 1));
			$buf .= substr($code, $a / 4, 1);
			$buf .= substr($code, ($a % 4) * 16, 1);
			$buf .= '==';
		}
	}
	close(FILE);
	return $buf;
}

sub FilePath
{
	my $url = shift;
	if ($url =~ /^(http|https|ftp|file):\/\//) {
		return $url;
	} elsif ($url =~ /^\//) {
		return $url;
	} else {
		return "$in_dir/$url";
	}
}

sub ExpandLinks
{
	my $text = shift;

	$text =~ s{
		<a\ href="url://([0-9]+)">
	}{
		my $url = "";
		if ($1 < scalar(@url_names)) {
			my $name = $url_names[$1];
			$url = $urls{$name} if $urls{$name};
		}
		"<a href=\"$url\">";
	}mgex;

	$text =~ s{
		<img\ src="([^"]*)"([^>]*)>
	}{
		my $url = $1;
		my $rest = $2;
		if ($url =~ /^url:\/\/([0-9]+)$/) {
			if ($1 < scalar(@url_names)) {
				my $name = $url_names[$1];
				$url = $urls{$name} if $urls{$name};
			}
		}
		my $img_text = "<img src=\"$url\"$rest>";
		if (not $disable_embedding) {
			if ($url =~ /\.(png|jpg|jpeg|gif)$/) {
				my $img_type = $1;
				my $src = Base64EncodeFile(FilePath($url));
				if ($src) {
					$img_text = "<img src=\"data:image/$img_type;base64,$src\"$rest>";
				}
			} elsif ($url =~ /\.svg$/) {
				my $svg_text = LoadFiles(FilePath($url));
				if ($svg_text) {
					$img_text = "<figure>$svg_text</figure>";
				}
			}
		}
		$img_text;
	}mgex;

	return $text;
}

sub GetTOC
{
	my $res = "  <nav class=\"toc\">
    <ul>
	  " . join("\n", map{"      $_"}@section_titles) . "
	</ul>
  </nav>";
	return $res;
}

sub Markdown
{
	my $text = shift;

	# standardize EOLs
	$text =~ s{\r\n}{\n}g; # DOS => Linux
	$text =~ s{\r}{\n}g;   # Mac => Linux

	# remove spaces in empty lines
	$text =~ s/^[ \t]+$//mg;

	# <tab> => <space> x 4
	$text =~ s{(.*?)\t}{$1.(' ' x (4 - length($1) % 4))}ge;

	$text = Sections($text);
	$text =~ s{(\<\/h1\>)}{
		my $toc = "$1
<section>
  " . GetTOC() . "
</section>
";
		$toc;
	}mex;
	$text = ExpandLinks($text);

	$text =~ s{$escape_table{'*'}}{*}g;
	$text =~ s{$escape_table{'_'}}{_}g;
	return $text;
}

sub LoadFiles
{
	my $text = "";
	while (my $file = shift) {
		print STDERR "Load file: $file\n" if $verbose;
		open INPUT, $file or die "Fail to read file '$file'";
		my @lines = <INPUT>;
		close INPUT;
		$text .= join("", @lines) . "\n";
	}
	return $text;
}

sub PrintUsage
{
	print "
Usage: " . basename($0) . " [options] <in.md> [out.html]

Options:
   -h / --help                     show this help message
   --css <style.css>               css files, can be specified repeatly
   --js <script.js>                javascript files, can be specified repeatly
   --disable-section-numbers       disable auto-increasing numbers on section titles
   --disable-toc                   disable table of content
   --disable-embedding             disable embedding css/js/image
   -v / --verbose                  show verbose messages
   -V / --version                  show version info
   --test                          self testing (for debugging)

";
	exit 1;
}

my @css = ();
my @js = ();
my $input = "";
my $output = "";

while (@ARGV) {
	my $opt = shift @ARGV;
	if ($opt eq "-h" or $opt eq "--help") {
		PrintUsage;
	} elsif ($opt eq "--css") {
		die "missing argument for $opt" unless @ARGV;
		push @css, shift @ARGV;
	} elsif ($opt eq "--js") {
		die "missing argument for $opt" unless @ARGV;
		push @js, shift @ARGV;
	} elsif ($opt eq "--disable-section-numbers") {
		$disable_section_numbers = 1;
	} elsif ($opt eq "--disable-toc") {
		$disable_toc = 1;
	} elsif ($opt eq "--disable-embedding") {
		$disable_embedding = 1;
	} elsif ($opt eq "-v" or $opt eq "--verbose") {
		++$verbose;
	} elsif ($opt eq "-V" or $opt eq "--version") {
		print "$version\n";
		exit 0;
	} elsif ($opt eq "--test") {
		unshift @INC, dirname($0);
		require("tests") or die;
		exit 0;
	} elsif ($opt =~ /^-/) {
		die "unexpected option '$opt'";
	} elsif (not $input) {
		$input = $opt;
	} elsif (not $output) {
		$output = $opt;
	} else {
		die "unexpected parameter '$opt'";
	}
}
PrintUsage unless $input;
$in_dir = dirname($input);
$output = "/dev/stdout" unless $output;

if (not @css and -f dirname($0) . "/default.css") {
	push @css, dirname($0) . "/default.css";
}

my $css = "";
if ($disable_embedding) {
	$css = join("\n", map{"<link rel=\"stylesheet\" type=\"text/css\" href=\"$_\">"}@css);
	$css .= "\n" if $css;
} else {
	$css = LoadFiles(@css);
	$css =~ s/\n+\z//g;
	$css = "<style>\n$css\n</style>\n" if $css;
}

my $js = "";
if ($disable_embedding) {
	$js = join("\n", map{"<script type=\"text/javascript\" src=\"$_\"></script>"}@js);
	$js .= "\n" if $js;
} else {
	$js = LoadFiles(@js);
	$js = "<script type=\"text/javascript\">\n<!--\n$js\n-->\n</script>" if $js;
}

my $text = Markdown(LoadFiles($input));

open OUTPUT, ">$output" or die "Fail to open output file '$output'";
print OUTPUT "<!DOCTYPE html>
<html>
<head>
<meta charset=\"UTF-8\">
<title>$article_title</title>
$css$js</head>
<body>
<article>
$text
</article>
</body>
</html>
";
close OUTPUT;

exit 0;
