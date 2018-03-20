CXXFLAGS+= -std=c++1z


all: as_ptr_const.html

%.html: %.md
	( echo "<html><body>" ; cmark $< ) > $@
