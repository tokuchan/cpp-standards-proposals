CXXFLAGS+= -std=c++1z


all: string_view string_view_conversion.html

%.html: %.md
	( echo "<html><body>" ; cmark $< ) > $@
