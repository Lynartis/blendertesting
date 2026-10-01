float4 main(PS_INPUT i) : SV_Target
{
float4 vPrev = texHistory.Sample(samplerHistory, i.vTex0 + UV_DISPLACEMENT);
uint uImpulses = 0;
float2 vTotalDir = float2(0.0f, 0.0f);

for (uint j = 0; j < NUM_IMPULSES; ++j)
{
float2 vDir = vWorld - GET_IMPULSE_POS(j);
float fSqrDist = 1.0f - saturate(dot(vDir, vDir) * GET_IMPULSE_RCP_RADIUS(j));
vTotalDir += vDir * fSqrDist;
uImpulses += (fSqrDist > 0.0f) ? 1 : 0;
}

float2 vRetDir;
float fDistance;
float fRetPower;

if (uImpulses > 0)
{
float2 vCenterOffset = vTotalDir / float(uImpulses);
vRetDir = TrailMap_PackDir(normalize(vCenterOffset));
fDistance = saturate(length(vCenterOffset) / MAX_IMPULSE_LENGTH);
fRetPower = 1.0f;
}
else
{
vRetDir = TrailMap_GetPackedDir(vPrev);
fDistance = saturate(vPrev.b - DECAY);
fRetPower = saturate(TrailMap_GetPower(vPrev) - DECAY);
}

float4 vRet;
vRet.xy = vRetDir;
vRet.b = fDistance;
vRet.a = fRetPower;
return vRet;
}






static const float MIN_POWER         = 0.01f;
static const float CENTER_EPSILON    = 0.05f;
static const float CENTER_EPSILON_SQ = CENTER_EPSILON * CENTER_EPSILON;


// ============================================================================
// HELPERS
// ============================================================================

bool HasImpulse(float3 sample)
{
    return sample.z > MIN_POWER;
}


float3 GetCenter(float3 samplePoint, float3 sample)
{
    return float3(
        samplePoint.xy + sample.xy,
        0.0f
    );
}


bool SameCenter(float3 A, float3 B)
{
    float2 delta = A.xy - B.xy;

    return dot(delta, delta) <= CENTER_EPSILON_SQ;
}


// ============================================================================
// TEST ONE CENTER
//
// A      = original 3D point
// X      = center we want to test: C, D1, D2...
// Source = point where X was discovered
//
// For C:
//      X      = C
//      Source = B
//
// For D1:
//      X      = D1
//      Source = point where D1 was found
//
// Returns true if X reaches A.
//
// If another impulse is found:
//      D1 / D2 are returned so we can test them afterwards.
// ============================================================================

bool TestCenter(
    float3 A,
    float3 X,
    float3 Source,

    out float3 D1,
    out float3 D1Source,
    out float  D1Power,

    out float3 D2,
    out float3 D2Source,
    out float  D2Power)
{
    D1 = 0.0f;
    D1Source = 0.0f;
    D1Power = 0.0f;

    D2 = 0.0f;
    D2Source = 0.0f;
    D2Power = 0.0f;


    // ------------------------------------------------------------------------
    // XA = full 3D vector from candidate X to original point A.
    //
    // R = how far X must reach to contain A.
    // ------------------------------------------------------------------------

    float3 XA = A - X;

    float R = length(XA);


    // ------------------------------------------------------------------------
    // Direction on Z=0 from X toward the point where X was discovered.
    // ------------------------------------------------------------------------

    float2 XS = Source.xy - X.xy;

    float xsLengthSq = dot(XS, XS);

    float2 XSDir;

    if (xsLengthSq > 1e-8f)
    {
        XSDir = XS * rsqrt(xsLengthSq);
    }
    else
    {
        XSDir = float2(1.0f, 0.0f);
    }


    // ========================================================================
    // FIRST SIDE
    //
    // P1 is exactly distance R from X.
    // ========================================================================

    float3 P1 = float3(
        X.xy + XSDir * R,
        0.0f
    );


    float3 sampleP1 = SamplePoint(P1);


    // ------------------------------------------------------------------------
    // Nothing.
    //
    // Simple first-version rule:
    // X is OUT.
    // ------------------------------------------------------------------------

    if (!HasImpulse(sampleP1))
    {
        return false;
    }
    else
    {
        float3 centerP1 = GetCenter(P1, sampleP1);


        // --------------------------------------------------------------------
        // P1 gives X.
        //
        // distance(X,P1) == distance(X,A)
        //
        // Therefore X reaches A.
        // --------------------------------------------------------------------

        if (SameCenter(centerP1, X))
        {
            return true;
        }
        else
        {
            // ----------------------------------------------------------------
            // P1 gives another center.
            //
            // Call it D1.
            // X may simply be hidden here.
            // ----------------------------------------------------------------

            D1 = centerP1;
            D1Source = P1;
            D1Power = sampleP1.z;


            // ================================================================
            // SECOND / OPPOSITE SIDE
            // ================================================================

            float3 P2 = float3(
                X.xy - XSDir * R,
                0.0f
            );


            float3 sampleP2 = SamplePoint(P2);


            // ---------------------------------------------------------------
            // Opposite side empty.
            //
            // X is OUT.
            // Keep D1 for later.
            // ---------------------------------------------------------------

            if (!HasImpulse(sampleP2))
            {
                return false;
            }
            else
            {
                float3 centerP2 = GetCenter(P2, sampleP2);


                // -----------------------------------------------------------
                // Opposite side gives X.
                //
                // X reaches A.
                // -----------------------------------------------------------

                if (SameCenter(centerP2, X))
                {
                    return true;
                }
                else
                {
                    // -------------------------------------------------------
                    // Opposite side gives something else.
                    //
                    // If it is NOT the same D1, call it D2.
                    // -------------------------------------------------------

                    if (!SameCenter(centerP2, D1))
                    {
                        D2 = centerP2;
                        D2Source = P2;
                        D2Power = sampleP2.z;
                    }


                    return false;
                }
            }
        }
    }
}


// ============================================================================
// MAIN SEARCH
//
// Explicitly:
//
//      B -> C
//      test C
//
//      if C fails and found D1:
//          test D1
//
//      if C fails and found D2:
//          test D2
//
// No loops yet.
// ============================================================================

bool FindTrailImpulse(
    float3 worldPos,
    out float3 impulseCenter,
    out float impulsePower)
{
    impulseCenter = 0.0f;
    impulsePower = 0.0f;


    // ========================================================================
    // A
    // ========================================================================

    float3 A = worldPos;


    // ========================================================================
    // B = projection of A onto Z = 0
    // ========================================================================

    float3 B = float3(
        A.xy,
        0.0f
    );


    // ========================================================================
    // SAMPLE B
    // ========================================================================

    float3 sampleB = SamplePoint(B);


    if (!HasImpulse(sampleB))
    {
        // B found nothing.
        return false;
    }
    else
    {
        // ====================================================================
        // C = B + offset stored in RG
        // ====================================================================

        float3 C = GetCenter(B, sampleB);

        float CPower = sampleB.z;


        // --------------------------------------------------------------------
        // Named vectors exactly matching the concept:
        // --------------------------------------------------------------------

        float3 BA = A - B;
        float3 BC = C - B;
        float3 CA = A - C;


        // ====================================================================
        // Variables for centers discovered while testing C.
        // ====================================================================

        float3 D1;
        float3 D1Source;
        float  D1Power;

        float3 D2;
        float3 D2Source;
        float  D2Power;


        // ====================================================================
        // TEST C
        //
        // C was discovered from B.
        // ====================================================================

        bool insideC = TestCenter(
            A,
            C,
            B,

            D1,
            D1Source,
            D1Power,

            D2,
            D2Source,
            D2Power
        );


        if (insideC)
        {
            impulseCenter = C;
            impulsePower = CPower;

            return true;
        }
        else
        {
            // =================================================================
            // C failed.
            //
            // If we found D1, test it.
            // =================================================================

            if (D1Power > MIN_POWER)
            {
                float3 E1;
                float3 E1Source;
                float  E1Power;

                float3 E2;
                float3 E2Source;
                float  E2Power;


                bool insideD1 = TestCenter(
                    A,
                    D1,
                    D1Source,

                    E1,
                    E1Source,
                    E1Power,

                    E2,
                    E2Source,
                    E2Power
                );


                if (insideD1)
                {
                    impulseCenter = D1;
                    impulsePower = D1Power;

                    return true;
                }
            }


            // =================================================================
            // D1 did not work.
            //
            // If C also discovered a different D2, test D2.
            // =================================================================

            if (D2Power > MIN_POWER)
            {
                float3 E1;
                float3 E1Source;
                float  E1Power;

                float3 E2;
                float3 E2Source;
                float  E2Power;


                bool insideD2 = TestCenter(
                    A,
                    D2,
                    D2Source,

                    E1,
                    E1Source,
                    E1Power,

                    E2,
                    E2Source,
                    E2Power
                );


                if (insideD2)
                {
                    impulseCenter = D2;
                    impulsePower = D2Power;

                    return true;
                }
            }


            // C, D1 and D2 did not confirm.
            return false;
        }
    }
}










A = worldPos
↓
B = (A.xy, 0)
↓
sample B
│
├─ nothing → return false
│
└─ C = B + sample.xy
   │
   ▼
   R = length(A - C)
   Dir = normalize(B - C)
   │
   ▼
   P1 = C + Dir * R
   │
   ▼
   sample P1
   │
   ├─ C
   │   └─ return C
   │
   ├─ nothing
   │   └─ C out
   │
   └─ D1
       │
       ▼
       P2 = C - Dir * R
       │
       ▼
       sample P2
       │
       ├─ C
       │   └─ return C
       │
       ├─ nothing
       │   └─ C out, keep D1
       │
       ├─ D1
       │   └─ C out, keep D1
       │
       └─ different center
           └─ C out, keep D1 + D2

Then:

TestCenter(A, D1, D1Source)
TestCenter(A, D2, D2Source)












// ============================================================================
// SIMPLE TRAILMAP 3D SEARCH - NO LOOPS
//
// SamplePoint(worldPoint) is provided externally:
//
//      float3 sample = SamplePoint(worldPoint);
//
//      sample.xy = direction + magnitude from sample point to impulse center
//                  (already non-normalized, so it is the complete XY offset)
//
//      sample.z  = impulse power
//
// Therefore:
//
//      center.xy = samplePoint.xy + sample.xy
//
// ============================================================================

static const float MIN_POWER          = 0.01f;
static const float CENTER_EPSILON     = 0.05f;
static const float CENTER_EPSILON_SQ  = CENTER_EPSILON * CENTER_EPSILON;


// ============================================================================
// DATA
// ============================================================================

struct Candidate
{
    uint   valid;

    float3 center;          // C, D1, D2...
    float3 discoveredFrom;  // B, B2, B3...
    float  power;
};


struct CandidateTest
{
    uint inside;

    // Other impulses found while testing this candidate.
    Candidate D1;
    Candidate D2;
};


struct SearchResult
{
    uint   found;

    float3 center;
    float  power;
};


// ============================================================================
// HELPERS
// ============================================================================

bool HasImpulse(float3 sample)
{
    return sample.z > MIN_POWER;
}


float3 GetSampleCenter(
    float3 samplePoint,
    float3 sample)
{
    // sample.xy already contains direction + magnitude.
    return float3(
        samplePoint.xy + sample.xy,
        0.0f
    );
}


bool SameCenter(
    float3 A,
    float3 B)
{
    float2 delta = A.xy - B.xy;

    return
        dot(delta, delta) <=
        CENTER_EPSILON_SQ;
}


Candidate MakeCandidate(
    float3 samplePoint,
    float3 sample)
{
    Candidate candidate = (Candidate)0;

    if (!HasImpulse(sample))
        return candidate;

    candidate.valid          = 1;
    candidate.center         = GetSampleCenter(samplePoint, sample);
    candidate.discoveredFrom = samplePoint;
    candidate.power          = sample.z;

    return candidate;
}


// ============================================================================
// TEST ONE CANDIDATE
//
// Example for C:
//
//      A      = original worldPos
//      X      = C
//      Source = B
//
//      XA = A - X
//      R  = length(XA)
//
//      XS = Source - X
//      Dir = normalize(XS.xy)
//
//      P1 = X + Dir * R
//
//      Sample P1:
//
//          X       -> A is inside X
//          Nothing -> X is outside
//          D1      -> test opposite side
//
//      P2 = X - Dir * R
//
//      Sample P2:
//
//          X       -> A is inside X
//          Nothing -> X outside, keep D1
//          D1      -> X outside, keep D1
//          D2      -> X outside, keep D1 + D2
//
// The exact same function works for D:
//
//      X      = D
//      Source = the point where D was discovered
//
// ============================================================================

CandidateTest TestCandidate(
    float3 A,
    Candidate X)
{
    CandidateTest result = (CandidateTest)0;

    if (!X.valid)
        return result;


    // ------------------------------------------------------------------------
    // XA
    //
    // Full 3D distance from candidate center X to original point A.
    // ------------------------------------------------------------------------

    float3 XA = A - X.center;

    float radiusToA =
        length(XA);


    // ------------------------------------------------------------------------
    // XS
    //
    // Direction on Z=0 from X toward the point where X was discovered.
    //
    // For C:
    //      X      = C
    //      Source = B
    //
    // For D1:
    //      X      = D1
    //      Source = B2
    // ------------------------------------------------------------------------

    float2 XS =
        X.discoveredFrom.xy -
        X.center.xy;

    float xsLengthSq =
        dot(XS, XS);

    float2 XSDir;

    if (xsLengthSq > 1e-8f)
        XSDir = XS * rsqrt(xsLengthSq);
    else
        XSDir = float2(1.0f, 0.0f);


    // ========================================================================
    // FIRST SIDE
    //
    // P1 is radiusToA away from X, along the X -> Source line.
    // ========================================================================

    float3 P1 =
        float3(
            X.center.xy +
            XSDir * radiusToA,
            0.0f
        );


    float3 sampleP1 =
        SamplePoint(P1);


    // ------------------------------------------------------------------------
    // Nothing at P1.
    //
    // First/simple rule:
    //      X does not reach A.
    // ------------------------------------------------------------------------

    if (!HasImpulse(sampleP1))
    {
        result.inside = 0;
        return result;
    }


    float3 centerP1 =
        GetSampleCenter(
            P1,
            sampleP1
        );


    // ------------------------------------------------------------------------
    // P1 gives X.
    //
    // distance(X,P1) == distance(X,A)
    //
    // Therefore X reaches A.
    // ------------------------------------------------------------------------

    if (SameCenter(centerP1, X.center))
    {
        result.inside = 1;
        return result;
    }


    // ------------------------------------------------------------------------
    // P1 gives another impulse.
    //
    // Call it D1.
    //
    // X may simply be hidden by D1, so test the opposite side.
    // ------------------------------------------------------------------------

    result.D1 =
        MakeCandidate(
            P1,
            sampleP1
        );


    // ========================================================================
    // OPPOSITE SIDE
    // ========================================================================

    float3 P2 =
        float3(
            X.center.xy -
            XSDir * radiusToA,
            0.0f
        );


    float3 sampleP2 =
        SamplePoint(P2);


    // ------------------------------------------------------------------------
    // Opposite side empty.
    //
    // X is considered outside.
    // D1 remains available.
    // ------------------------------------------------------------------------

    if (!HasImpulse(sampleP2))
    {
        result.inside = 0;
        return result;
    }


    float3 centerP2 =
        GetSampleCenter(
            P2,
            sampleP2
        );


    // ------------------------------------------------------------------------
    // Opposite side gives X.
    //
    // X reaches A.
    // ------------------------------------------------------------------------

    if (SameCenter(centerP2, X.center))
    {
        result.inside = 1;
        return result;
    }


    // ------------------------------------------------------------------------
    // Opposite side gives something else.
    //
    // If it is the same D1, don't store it twice.
    //
    // Otherwise call it D2.
    // ------------------------------------------------------------------------

    if (!SameCenter(centerP2, result.D1.center))
    {
        result.D2 =
            MakeCandidate(
                P2,
                sampleP2
            );
    }


    result.inside = 0;

    return result;
}


// ============================================================================
// COMPLETE FIRST VERSION - NO LOOPS
//
// We explicitly do:
//
//      B -> C
//      test C
//
//      if needed:
//          test D1
//
//      if needed:
//          test D2
//
// New candidates discovered while testing D1/D2 are intentionally NOT
// followed yet. Adding that repetition is the next step where this becomes
// a tiny queue/loop.
// ============================================================================

SearchResult FindTrailImpulse(
    float3 worldPos)
{
    SearchResult result = (SearchResult)0;


    // ========================================================================
    // A
    // Original 3D point.
    // ========================================================================

    float3 A =
        worldPos;


    // ========================================================================
    // B
    // Project A vertically onto Z = 0.
    // ========================================================================

    float3 B =
        float3(
            A.xy,
            0.0f
        );


    // ========================================================================
    // SAMPLE B
    // ========================================================================

    float3 sampleB =
        SamplePoint(B);


    // ------------------------------------------------------------------------
    // B gives nothing.
    // ------------------------------------------------------------------------

    if (!HasImpulse(sampleB))
    {
        result.found = 0;
        return result;
    }
    else
    {
        // ====================================================================
        // B gives C.
        //
        // C = B + sampleB.xy
        // ====================================================================

        Candidate C =
            MakeCandidate(
                B,
                sampleB
            );


        // Useful conceptual vectors:
        //
        // BA = A - B
        // BC = C - B
        // CA = A - C

        float3 BA =
            A - B;

        float3 BC =
            C.center - B;

        float3 CA =
            A - C.center;


        // ====================================================================
        // TEST C
        // ====================================================================

        CandidateTest testC =
            TestCandidate(
                A,
                C
            );


        // --------------------------------------------------------------------
        // C reaches A.
        // --------------------------------------------------------------------

        if (testC.inside)
        {
            result.found  = 1;
            result.center = C.center;
            result.power  = C.power;

            return result;
        }
        else
        {
            // =================================================================
            // C did not confirm.
            //
            // Did testing C discover D1?
            // =================================================================

            if (testC.D1.valid)
            {
                Candidate D1 =
                    testC.D1;


                // Conceptually:
                //
                // DA = A - D1
                //
                // D1.discoveredFrom is automatically the point
                // where D1 was found.

                float3 D1A =
                    A - D1.center;


                CandidateTest testD1 =
                    TestCandidate(
                        A,
                        D1
                    );


                // -------------------------------------------------------------
                // D1 reaches A.
                // -------------------------------------------------------------

                if (testD1.inside)
                {
                    result.found  = 1;
                    result.center = D1.center;
                    result.power  = D1.power;

                    return result;
                }
            }


            // =================================================================
            // D1 didn't confirm or didn't exist.
            //
            // Test D2 if C discovered a second different candidate.
            // =================================================================

            if (testC.D2.valid)
            {
                Candidate D2 =
                    testC.D2;


                float3 D2A =
                    A - D2.center;


                CandidateTest testD2 =
                    TestCandidate(
                        A,
                        D2
                    );


                // -------------------------------------------------------------
                // D2 reaches A.
                // -------------------------------------------------------------

                if (testD2.inside)
                {
                    result.found  = 1;
                    result.center = D2.center;
                    result.power  = D2.power;

                    return result;
                }
            }


            // =================================================================
            // C, D1 and D2 all failed.
            // =================================================================

            result.found = 0;

            return result;
        }
    }
}
















A = worldPos
↓
B = (A.xy, 0)
↓
sampleB = SamplePoint(B)

IF sampleB is empty
    RETURN NONE

ELSE
    C = B + sampleB.xy
    C.power = sampleB.z

    TEST C
        R = distance(C, A)
        Dir = normalize(B - C)

        P1 = C + Dir * R
        sample P1

        IF P1 == C
            RETURN C

        ELSE IF P1 is empty
            C is OUT

        ELSE
            D1 = P1 + sampleP1.xy

            P2 = C - Dir * R
            sample P2

            IF P2 == C
                RETURN C

            ELSE IF P2 is empty
                C is OUT
                keep D1

            ELSE IF P2 == D1
                C is OUT
                keep D1

            ELSE
                D2 = P2 + sampleP2.xy
                C is OUT
                keep D1 + D2


    IF D1 exists
        TEST D1 with exactly the same function
        IF inside
            RETURN D1

    IF D2 exists
        TEST D2 with exactly the same function
        IF inside
            RETURN D2

RETURN NONE
