---
layout: page
subheadline: "Computational Molecular Biology"
title: "Sequence alignment [C++]"
teaser: "Basic sequence alignment in C++ using the Needleman and Wunsch-algorithm."
header:
    image_fullwidth: "header_R.jpg"
image:
    thumb: Needleman-Wunsch_PW.png
categories:
    - blog
tags:
    - Blog
    - C++
    - Computational Molecular Biology
    - Sequence Alignment
    - Multiple Sequence Alignment
comments: true
show_meta: true
---


## Introduction

Hello all, after some time I decided to create another post! My [last post](/blog/Poetry/) was about the analysis of 119 scraped poems, finding differences and similarities between different kinds of poems. This time, I will talk you through something more applicable to my own field of study: the basic alignment of two DNA sequences using the Needleman and Wunsch-algorithm. The reason I chose this subject is because it is my current subject on a course I'm following and I wanted to show what I am doing.
This is also the first time on this blog that I'll publish C++ code instead of R-script, as this was insisted on by my course supervisor.

The Needleman and Wunsch-algorithm could be seen as one of the basic global alignment techniques: it aligns two sequences using a scoring matrix and a traceback matrix, which is based on the prior. Many other, way more complex algorithms have been written since the publication of this algorithm, but it is a good basis for more complicated problems and solutions. The code that I wrote is based on the script found on [this site](http://www.rolfmuertter.com/code/nw.php). Some parts of it will remain mostly unchanged.

*All data (anonymised if applicable), used script and created visualisations can be found in my [GitHub repository]({{ site.githublink }}).*

I love visualising interesting, uncommon datasets and working on projects so please [let me know](#disqus_thread) down in the comments if you have any request or interesting data laying around!

## Needleman and Wunsch-algorithm

I won't go into too much detail on how this algorithm works, but the basics will be covered along the way.
The principles are: a perfect match on a position on two sequences gives the highest score, mismatches get penalties and gaps are usually penalised using some function that takes into account how long the gap will actually be among multiple positions.

![Image not found!](/images/Cpp/2018-03-21_Needleman-Wunsch_pairwise_sequence_alignment.png){:target="_blank" .image-wrapper}
    
<p class="image-caption">Needleman-Wunsch pairwise sequence alignment. <a href="https://commons.wikimedia.org/w/index.php?curid=31963972">By Slowkow - Own work, CC0</a></p>

The scoring matrix shown above show the maximal alignment score for any given sequence alignment at that point. To get the optimal alignment, you would follow the highest scoring cells from the lower-right corner to the upper-left corner. Then you invert the given sequence to get the actual alignment. Because it aligns from the back to the front, some alignments might be somewhat unexpected as you actually read it from left to right while it is aligned from right to left.

## Writing the algorithm

To start off with, put this on the top of you c++ script:

```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <cmath>
#include <algorithm>
#include <time.h>

using namespace std;
```

Some functions will need these to be executed.

### Scoring methods

First things first, we'll determine the scoring method that will be used. This will be done by using a substitution scoring matrix in which all possible combinations are given a certain score:

|         |  C  |  T  |  A  |  G  |
|:-------:|:---:|:---:|:---:|:---:|
|  **C**  |  5  |  -2 |  -5 |  -5 |
|  **T**  |  -2 |  5  |  -5 |  -5 |
|  **A**  |  -5 |  -5 |  5  |  -2 |
|  **G**  |  -5 |  -5 |  -2 |  5  |

The reason for the -2 in A-G and C-T matching is because purine-purine (A/G) and pyrimidine-pyrimidine (C/T) mutations are biologically seen more common than other mutations. In combination with the affine gap function, the algorithm might prefer a longer gap over a gap interrupted by one of these mutations.

The gap penalty will be calculated by the function ```gap_affinity()```, which defines an "affine gap penalty": the initial gap penalty will be higher with every directly following gap receiving a lower penalty. This ensures the algorithm to favor longer gaps over multiple singular gaps, which is also biologically seen more realistic.

```cpp
int gap_affinity (int gap, int gap_ext, int& length){
    int gap_aff = gap + (gap_ext * length);

    return gap_aff;
}
```

### Overview structure

Next, we'll go through the code from the bottom to the top. This is done for a more readable structure of the code.
The ```main()``` will call all main functions, gather user input and output some feedback:

```cpp
int main (int argc, char** argv){
    string A, B; // sequence to be aligned A and B
    string A_al, B_al = ""; // aligned sequence A and B
    bool print_mat; // whether to print matrices etc or just the aligned sequence
    bool print_align; // if alignments should be printed
    int align_nuc = 150; // amount of nucleotides per row
    int a = 5; // match
    int b = -2; // purine-purine / pyrimidine-pyrimidine
    int c = -5; // mismatch
    int gap = 2; // initial gap penalty. Gap penalty is lower than mismatch: two sequences from same species assumed.
    float gap_ext = 1; // bigger gap penalties for affine gap penalty
    string line; // reading in data
      
    // User interface
    cout << endl << "Print matrices? [0/1]" << endl;
    cin >> print_mat;
    cout << "Print alignments? [0/1]" << endl;
    cin >> print_align;
    cout << "Nucleotides per row? (150 recommended)" << endl;
    cin >> align_nuc;

    cout << "==============================" << endl;

    // Read in the alignments
    ifstream myfile ("sequences.txt");
    if (myfile.is_open()){
        getline (myfile,line);
        A = line;
        A.erase( A.end()-1 ); // remove whitespace at end
        getline (myfile,line);
        B = line;
        myfile.close();
    }else{
        cout << "Unable to open file";
        return 1;
    }

    // Run the alignment script
    int A_n = A.length();
    int B_n = B.length();

    NW (A, B, A_al, B_al, A_n, B_n, a, b, c, gap, gap_ext, print_align, print_mat, align_nuc);

    // Output parameters
    cout << endl << "Used parameters:" << endl;
    cout << "Match = " << a << endl;
    cout << "Mismatch = " << c << endl;
    cout << "Purine/purine or pyrimidine/pyrimidine = " << b << endl;
    cout << "Gap = -" << gap << endl;
    cout << "Extended gap = -" << gap_ext << endl;

    return 0;
}
```

As you can see, several variables are initialised and the sequences A and B will be read in from a file called "*sequences.txt*". ```NW()``` is the algorithm which will align the sequences. Let's head to this function and see what it does!

First, the required matrices are created and then initialised by ```init()```. The horizontal axis will cover sequence A and the vertical axis sequence B. After this, the ```alignment()``` function will actually align the two sequences *from the back to the front*: this will sometimes result in a somewhat counterintuitive global alignment. The execution time is being calculated for evaluation purposes, but can easily be skipped by editing it yourself. 

After the calculations, the matrices and alignment are optionally printed and the results are exported together with the used parameters. Some print/export operations are not written in a seperate function and the reason is quite simple: I sadly didn't have the time to convert them in a function.

Printing of the alignment is done using a more biologically relevant method: nucleotides *n* through *n+align_nuc* of aligned sequences A and B are printed below each other iteratively, giving more insight in the alignment.

```cpp
// Initiate matrices, align and export
int** NW (string A, string B, string& A_al, string& B_al, int A_n, int B_n, int a, int b, int c, int gap, int gap_ext, bool print_align, bool print_mat, int align_nuc){
    // Create alignment matrix
    int** M = new int* [B_n+1];
    for( int i = 0; i <= B_n; i++ ){
        M[i] = new int [A_n+1];
    }

    // Create traceback matrix
    char** M_tb = new char* [B_n+1];
    for( int i = 0; i <= B_n; i++ ){
        M_tb[i] = new char [A_n+1];
    }

    clock_t t; // for timing execution
    t = clock(); // get time of start

    // Initialize traceback and F matrix (fill in first row and column)
    init (M, M_tb, A_n, B_n, gap, gap_ext);


    // Create alignment
    alignment (M, M_tb, A, B, A_al, B_al, A_n, B_n, a, b, c, gap, gap_ext);

    t = clock() - t; // get time when finished
    int score = M[B_n][A_n]; // get alignment score

    if(print_mat == 1){
        print_mtx(M, A, B, A_n, B_n);
        print_tb(M_tb, A, B, A_n, B_n);
    }

    if(print_align == 1){
        cout << endl << "Alignments:" << endl;
        int start = 0; // start of new line for printing alignments
        int cntr = 0; // iterator for printing alignments
        int Al_n = A_al.length(); // length of alignment
        do{
            cout << start+1 << " A: ";
            for (cntr = start; cntr < start+align_nuc; cntr++){
                if(cntr < Al_n){
                    cout << A_al[cntr];
                }else{
                    break;
                }
            }
            cout << " " << cntr << endl << start+1 << " B: ";
            for (cntr = start; cntr < start+align_nuc; cntr++){
                if(cntr < Al_n){
                    cout << B_al[cntr];
                }else{
                    break;
                }
            }
            cout << " " << cntr << endl << endl;
            start += align_nuc;
        }while(start <= Al_n);
    }

    // Show score and runtime
    cout << "Alignment score: " << score << endl;
    cout << "Alignment took " << t << "clicks (" << (t)/CLOCKS_PER_SEC << "seconds).\n";

    // Export alignment to file
    cout << "Exporting results..." << endl;
    ofstream myfile;
    myfile.open ("alignment.txt");
    myfile << "Used parameters:\n";
    myfile << "Match = " << a << "\n";
    myfile << "Mismatch = " << c << "\n";
    myfile << "Purine/purine or pyrimidine/pyrimidine = " << b << "\n";
    myfile << "Gap = -" << gap << "\n";
    myfile << "Extended gap = -" << gap_ext << "\n";
    myfile << "Average elapsed time = " << t << " - " << (t)/CLOCKS_PER_SEC << " [clicks - seconds]." << endl << endl;
    myfile << "Input A:\n" << A << "\n";
    myfile << "Input B:\n" << B << "\n";
    myfile << "\nAlignment:\n";
    int start = 0; // start of new line for printing alignments
    int cntr = 0; // iterator for printing alignments
    int Al_n = A_al.length(); // length of alignment
    do{
        myfile << start+1 << " A: ";
        for (cntr = start; cntr < start+align_nuc; cntr++){
            if(cntr < Al_n){
                myfile << A_al[cntr];
            }else{
                break;
            }
        }
        myfile << " " << cntr << "\n" << start+1 << " B: ";
        for (cntr = start; cntr < start+align_nuc; cntr++){
            if(cntr < Al_n){
                myfile << B_al[cntr];
            }else{
                break;
            }
        }
        myfile << " " << cntr << "\n\n";
        start += align_nuc;
    }while(start <= Al_n);
    myfile << "Alignment score: " << score << "\n";
    myfile.close();

    // Export scoring matrix to file
    myfile.open ("mtx_scoring.txt");
    myfile << "-\t-\t";
    for( int j = 0; j < A_n; j++ ){
        myfile << A[j] << "\t";
    }
    myfile << "\n-\t";
    for( int i = 0; i <= B_n; i++ ){
        int j = 0;
        if( i > 0){
            myfile << B[i-1] << "\t";
        }
        do{
            myfile << M[i][j] << "\t";
            j++;
        }while(j < A_n);
        myfile << M[i][j] << "\n";
    }
    myfile.close();

    // Export traceback matrix to file
    myfile.open ("mtx_trace.txt");
    myfile << "-\t-\t";
    for( int j = 0; j < A_n; j++ ){
        myfile << A[j] << "\t";
    }
    myfile << "\n-\t";
    for( int i = 0; i <= B_n; i++ ){
        int j = 0;
        if( i > 0){
            myfile << B[i-1] << "\t";
        }
        do{
            myfile << M_tb[i][j] << "\t";
            j++;
        }while(j < A_n);
        myfile << M_tb[i][j] << "\n";
    }
    myfile.close();
    cout << "Results exported!" << endl;

    // Free memory
    for( int i = 0; i <= B_n; i++ )  delete M[i];
    delete[] M;
    for( int i = 0; i <= B_n; i++ )  delete M_tb[i];
    delete[] M_tb;

    return 0;
}
```

The ```init()``` function already creates the first column and row, filling them with a hardcoded affine gap function. The traceback matrix automatically gets a horizontal or vertical line, indicitating a step left or up (both gaps, but in either B or A respectively). The rest of the matrix will be filled by the ```alignment()``` function, which is discussed next.

```cpp
// Initialise scoring matrix with first row and column
void  init (int** M, char** M_tb, int A_n, int B_n, int gap, int gap_ext){
    M[0][0] =  0;
    M_tb[0][0] = 'n';

    int i=0, j=0;

    for( j = 1; j <= A_n; j++ ){
        M[0][j] = - ( gap + ( gap_ext * (j - 1) ) ); // manually apply affine gap
        M_tb[0][j] =  '-';
    }
    for( i = 1; i <= B_n; i++ ){
        M[i][0] = - ( gap + ( gap_ext * (i - 1) ) ); // manually apply affine gap
        M_tb[i][0] =  '|';
    }
}
```

### Aligning the sequences

The alignment is made by the function ```alignment()```, which also takes the gap penalty as variable to feed into the affine gap function. First, the algorithm scores all possible alignment possibilities in the scoring matrix using the substitution scoring matrix. Secondly, the traceback matrix is created, based on the scoring matrix: From the lower-right corner, the highest scoring cell directly above, diagonally left or left indicates the next step in the alignment up to the upper left corner. This will lead to the final, most optimal global alignment. Lastly, the traceback matrix is used to create the *backward* alignments: it follows the traceback along the highest scoring cells from the lower-right corner to the upper-left corner, introducing gaps in sequence A or B when not going diagonal.

```cpp
// Needleman and Wunsch algorithm
int alignment (int** M, char** M_tb, string A, string B, string& A_al, string& B_al, int A_n, int B_n, int a, int b, int c, int gap, int gap_ext){
    int x = 0, y = 0;
    int scU, scD, scL; // scores for respectively cell above, diagonal and left
    char ptr, nuc;
    int i = 0, j = 0;
    int length = 0; // initial gap length

    // create substitution scoring matrix
    const int  s[4][4] =   {
                            { a, b, c, c },
                            { b, a, c, c },
                            { c, c, a, b },
                            { c, c, b, a }
                           };

    for( i = 1; i <= B_n; i++ ){
        for( j = 1; j <= A_n; j++ ){
            nuc = A[j-1];

            switch( nuc )
            {
                case 'C':  x = 0;  break;
                case 'T':  x = 1;  break;
                case 'A':  x = 2;  break;
                case 'G':  x = 3;
            }

            nuc = B[i-1];

            switch( nuc )
            {
                case 'C':  y = 0;  break;
                case 'T':  y = 1;  break;
                case 'A':  y = 2;  break;
                case 'G':  y = 3;
            }

            scU = M[i-1][j] - gap_affinity(gap, gap_ext, length); // get score if trace would go up
            scD = M[i-1][j-1] + s[x][y]; // get score if trace would go diagonal
            scL = M[i][j-1] - gap_affinity(gap, gap_ext, length); // get score if trace would go left

            M[i][j] = max_score (scU, scD, scL, &ptr, length); // get max score for current optimal global alignment

            M_tb[i][j] = ptr;
        }
    }
    i--; j--;

    while( i > 0 || j > 0 ){
        switch( M_tb[i][j] )
        {
            case '|' :      A_al += '-';
                            B_al += B[i-1];
                            i--;
                            break;

            case '\\':      A_al += A[j-1];
                            B_al += B[i-1];
                            i--;  j--;
                            break;

            case '-' :      A_al += A[j-1];
                            B_al += '-';
                            j--;
        }
    }

    reverse( A_al.begin(), A_al.end() );
    reverse( B_al.begin(), B_al.end() );

    return  0 ;
}
```

For the printing of the matrices, two functions were used from the original, referenced code:

```cpp
// Print the scoring matrix
void  print_mtx (int** M, string A, string B, int A_n, int B_n){
    cout << "        ";
    for( int j = 0; j < A_n; j++ ){
        cout << A[j] << "   ";
    }
    cout << "\n  ";

    for( int i = 0; i <= B_n; i++ ){
        if( i > 0 ){
            cout << B[i-1] << " ";
        }
        for( int j = 0; j <= A_n; j++ ){
            cout.width( 3 );
            cout << M[i][j] << " ";
        }
        cout << endl;
    }
    cout << endl;
}

// Print the traceback matrix
void  print_tb (char** M_tb, string A, string B, int A_n, int B_n){
    cout << "        ";
    for( int j = 0; j < A_n; j++ ){
        cout << A[j] << "   ";
    }
    cout << "\n  ";

    for( int i = 0; i <= B_n; i++ ){
        if( i > 0 ){
            cout << B[i-1] << " ";
        }
        for( int j = 0; j <= A_n; j++ ){
            cout.width( 3 );
            cout << M_tb[i][j] << " ";
        }
        cout << endl;
    }
    cout << endl;
}
```

This concludes all the necessary code for this script. The choice to export your files and time the execution duration is yours and the code for this is easily rewritten/deleted! This script can only be used for pairwise alignments, but stay tuned for an adaption of this script for a Multiple Sequence Alignment (MSA)!

*For the complete script, please see my [GitHub repository]({{ site.githublink }})*.
